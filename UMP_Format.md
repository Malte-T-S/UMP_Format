# UMP Format

The UMP format is used by YouTube for a number of requests and responses. This document details the format for the purposes of interoperability.

## Variable sized integers

The UMP format uses variable size integers in a number of places. These are implemented very similarly to variable length integers in RFC8794, with a slight variation in the case of 5-byte integers.

The first 5 bits of the first byte set the size of the integer:

- If the top bit is unset, it's a 1-byte value.
- If the top bit is set, but the next bit is not, it's a 2-byte value.
- If the top two bits are set, but the next bit is not, it's a 3-byte value.
- If the top three bits are set, but the next bit is not, it's a 4-byte value.
- If the top four bits are set, but the next bit is not, it's a 5-byte value.
- If all top five bits are set, the integer is invalid.

Getting the size of the integer from the first byte can be implemented as follows:

```c
int getVarIntSize(uint8_t b)
{
    int size = 0;
    for (int shift = 1; shift <= 5; shift++)
    {
        if ((b & (128 >> (shift - 1))) == 0)
        {
            size = shift;
            break;
        }
    }
    assert(size >= 1 && size <= 5);
    return size;
}
```

The remainder of the bits in the first byte are used as part of the integer, except for in 5-byte integers where those bits are ignored.

The variable integer decoding can be implemented as follows:

```c
int readNextByte(uint8_t* buf, int* pos)
{
    int ofs = *pos;
    *pos = ofs + 1;
    return buf[ofs];
}

int readVarInt(uint8_t* buf, int ofs)
{
    int pos = ofs;
    uint8_t prefix = readNextByte(buf, &ofs);
    int size = getVarIntSize(prefix);
    switch (size)
    {
        case 1:
            return prefix;
        case 2:
            return (readNextByte(buf, &pos) << 6) | (prefix & 0b111111);
        case 3:
            return
                (
                	readNextByte(buf, &pos) |
	                (readNextByte(buf, &pos) << 8)
            	) | (prefix & 0b11111);
        case 4:
            return
                (
                	readNextByte(buf, &pos) |
	                (readNextByte(buf, &pos) << 8) |
	                (readNextByte(buf, &pos) << 16)
            	) | (prefix & 0b1111);
        default:
            return
                (
                	readNextByte(buf, &pos) |
	                (readNextByte(buf, &pos) << 8) |
	                (readNextByte(buf, &pos) << 16) |
	                (readNextByte(buf, &pos) << 24)
            	);
    }
}
```

Note that the 5-byte integer case behaves differently, ignoring the bottom 3 bits in the first byte entirely, and just reading a 32-bit little-endian integer from the next four bytes. In the RFC8794 standard, a 5-byte integer includes the bottom 3 bits of the first byte, producing a 35-bit integer. Presumably this deviation was made so that the result could be neatly stored in a 32-bit integer, rather than needing to promote variables to 64-bit everywhere.

## High level structure

The UMP format requires that the Content-Type header in the HTTP response contains "application/vnd.yt-ump".

A UMP response is split into parts. Each part is prefixed by a pair of variable length integers, the first being the part type and the second being the part payload length. In pseudo-C, it'd look like this:

```c
struct UmpPart
{
    varInt type;
    varInt size;
    uint8_t data[size];
};
```

Note that you **must** treat the type field as a variable length integer. The current type numbers are all below 128, which will produce a single-byte encoding, but if you read the type ID as a single byte instead of properly decoding it as a variable length integer your implementation will break if/when new part types are added.

Each videoplayback HTTP response **usually** starts with the parts `STREAM_PROTECTION_STATUS` (type 58), `PLAYBACK_START_POLICY` (type 47), `REQUEST_IDENTIFIER` (type 52), `REQUEST_CALCELLATION_POLICY` (type 53), `NEXT_REQUEST_POLICY` (type 35) following any number of other parts (sometimes even no further content).

### Media Parts

When Media is contained in the response the actual media (`MEDIA` type 21) is **usually** preceded by a `MEDIA_HEADER` part (type 21) and followed by `MEDIA_END` (type 22). The `FORMAT_INITIALIZATION_METADATA` (type 42) is send at irregular intervals.

### Partial parts

Parts are not guaranteed to be wholly contained within one response payload. It is quite common to find that a part's length exceeds the length of the HTTP response payload. This means that the part will continue in the next response payload.

If a response payload is self-contained, i.e. it does not end with a partial part, the last part in the buffer will end precisely at the end of the buffer. Typically the last part will be `MEDIA_END` (type 22) with a single null byte payload, but this is not guaranteed.

If a payload is not self-contained, i.e. its final part has a length exceeding the amount of remaining data in the buffer, its data will continue in the next response payload. In such a case, the next payload will start with a onesie header part (type 20) followed by a part of the same type as the partial one, whose data is a continuation of the partial part from the previous payload. Parts can be split up over an arbitrary number of response payloads.

This is a little hard to picture, so here's an example. Let's say you've got a part with a length of 2,500,000 bytes, and each response payload can be maximum of 1MB (this is just for example; in practice there is no such hard limit). The resulting response payloads will look something like this:

```
response 1:
	part 20 (onesie header)
		size=...
		data=...
	part 21 (media data)
		size=2500000
		data=... (len=1000000)

response 2:
	part 20 (onesie header)
		size=...
		data=...
	part 21 (media data)
		size=1500000
		data=... (len=1000000)

response 3:
	part 20 (onesie header)
		size=...
		data=...
	part 21 (media data)
		size=500000
		data=... (len=500000)
	part 22 (MEDIA_END)
		size=1
		data=00
```

This gets decoded as a single type 21 part of size 2,500,000 bytes, followed by a type 22 part. Note that there could be a different part type in response 3, after part 21, instead of `MEDIA_END`. The end of a part is determined solely by all of its data being read; the next part type is irrelevant.

Once a partial part begins, responding with a different part type (e.g. sending a partial part 22, then following up with a part 32 before sending the rest of the first part) has undefined behaviour. As far as I could tell from the implementations, error handling is variable here. Some will throw an exception, but some appear to blindly accept the data as a continuation of the data, even if the part type ID is wrong. Fun! I would recommend being rigorous in checking for the correct type when decoding partial parts.

### Reading UMP parts

You can read UMP parts with a state machine:

1. Read a response payload in chunks.
2. If the amount of data remaining in the buffer is zero, go back to step 1.
3. Read the UMP part type as a variable sized integer.
4. Read the UMP part size as a variable sized integer.
5. If the part size is zero, decode the part as a zero-length part (i.e. no payload), passing in an empty buffer, then go back to step 2.
6. If the part size is less than or equal to the remaining buffer size, decode the part from that slice of the buffer, then increment the position within the buffer and go back to step 3.
7. If the part size is greater than the remaining buffer size, keep a copy of the remaining buffer data, read the next buffer, stitch the two together, and continue parsing from step 5.

## Part types

The following part types have been observed.

### Part 10: ONESIE_HEADER

OnesieHeader. Possibly older format, deprecated?

Unknown format, but probably protobufs.

### Part 11: ONESIE_DATA

If present, must be preceded by OnesieHeader (part 10). Possibly older format, deprecated?

Unknown format, but probably protobufs.

### Part 12: ONESIE_ENCRYPTED_MEDIA

Unknown format, but probably protobufs.

### Part 20: MEDIA_HEADER

Present at the start of all known UMP response payloads.

The payload is protobufs. So far I've been decoding this manually with https://protobuf-decoder.netlify.app/

- Field 1 is a varint and counts up each header per response.
- Field 2 is the video ID as a string.
- Field 3 is a varint which matches the `itag` URL parameter (parameters do not seem to match any of the itags on https://tyrrrz.me/blog/reverse-engineering-youtube).
- Field 4 is varint that matches the `lmt` URL parameter, which seems to be a microsecond timestamp related to the last modified time of the stream (likely the time at which encoding completed for a given video stream)
- Field 6 is a varint that is probably the start of the data range
- Field 13 is a protobuf message with two fields:
  - Field 1 is the `itag`
  - Field 2 is the `lmt` for the itag.
- Field 14 is the content length (without the leading null byte).

Field 14 probably represents the size of the media chunk being sent. If the payload does not end with a partial part, this number has always been observed to match the size of the data in the `MEDIA` part, not including the null byte prefix. When there is a partial part, the number tends to be a fair bit bigger, possibly representing the size of the entire video or chapter. When the video is a livestream, this value is missing, since YouTube has no idea when the stream will end.

### Part 21: MEDIA

Contains the actual media itself. Starts with a single null byte, followed by the media data.

If you pull the data out of these and into a webm file, you can play them with VLC!

### Part 22: MEDIA_END

Terminator part. Usually included at the end of all payloads where there is no partial payload at the end.

Size is usually 1, with the data being a single null byte.

### Part 31: LIVE_METADATA

Also referred to as "SABR Live Metadata", and is related to "SABR Live Protocols".

Possibly streaming related?

Unknown format.

### Part 33: LIVE_METADATA_PROMISE

SABR Live Metadata Promise, related to "SABR Live Protocols".

Possibly streaming related?

Unknown format.

### Part 34: LIVE_METADATA_PROMISE_CANCELLATION

Cancellation of a SABR Live Metadata Promise (originally sent as part 33), related to "SABR Live Protocols".

Possibly streaming related?

Unknown format.

### Part 35: NEXT_REQUEST_POLICY

Unknown purpose and format.

### Part 36: USTREAMER_VIDEO_AND_FORMAT_DATA

Probably related to signalling the media format information for livestreams.

Unknown format.

### Part 37: FORMAT_SELECTION_CONFIG

Format selection config. Related to the user changing format preferences (e.g. force 1080p).

Unknown format.

### Part 38: USTREAMER_SELECTED_MEDIA_STREAM

Clearly streaming related but not sure what this is for.

Unknown format.

### Part 39: ???

Not yet observed.

### Part 40: ???

Not yet observed.

### Part 41: ???

Not yet observed.

### Part 42: FORMAT_INITIALIZATION_METADATA

Contains information about the transmitted video.

- Field 1 is the video ID as a string.
- Field 2 is a protobuf containing two field equal to the fields 3 and 4 of the `MEDIA_HEADER` (type 20).
- Field 3 is a varint.
- Field 4 is a varint.
- Field 5 is a string and contains information about the data type (e.g. video/webm) and codec.
- Field 6 is a protobuf.
  - Field 6-1 is a varint.
  - Field 6-2 is a varint.
- Field 7 is a protobuf.
  - Field 7-1 is a varint.
  - Field 7-2 is a varint.
- Field 9 is a varint.
- Field 10 is a varint.

### Part 43: SABR_REDIRECT

Unknown purpose and format.

### Part 44: SABR_ERROR

Unknown purpose and format.

### Part 45: SABR_SEEK

Unknown purpose and format.

### Part 46: RELOAD_PLAYER_RESPONSE

Likely tells the player that the page needs to be reloaded.

Unknown format.

### Part 47: PLAYBACK_START_POLICY

Unknown purpose and format.

### Part 48: ALLOWED_CACHED_FORMATS

Unknown purpose and format.

### Part 49: START_BW_SAMPLING_HINT

Unknown purpose and format.

### Part 50: PAUSE_BW_SAMPLING_HINT

Unknown purpose and format.

### Part 51: SELECTABLE_FORMATS

Unknown purpose and format.

### Part 52: REQUEST_IDENTIFIER

Unknown purpose and format.

### Part 53: REQUEST_CANCELLATION_POLICY

Unknown purpose and format.

### Part 54: ONESIE_PREFETCH_REJECTION

Unknown purpose and format.

### Part 55: TIMELINE_CONTEXT

Unknown purpose and format.

### Part 56: REQUEST_PIPELINING

Unknown purpose and format.

### Part 57: SABR_CONTEXT_UPDATE

Unknown purpose and format.

### Part 58: STREAM_PROTECTION_STATUS

Unknown purpose and format.

### Part 59: SABR_CONTEXT_SENDING_POLICY

Unknown purpose and format.

### Part 60: LAWNMOWER_POLICY

Unknown purpose and format.

### Part 61: SABR_ACK

Unknown purpose and format.

### Part 62: END_OF_TRACK

Unknown purpose and format.

### Part 63: CACHE_LOAD_POLICY

Unknown purpose and format.

### Part 64: LAWNMOWER_MESSAGING_POLICY

Unknown purpose and format.

### Part 65: PREWARM_CONNECTION

Unknown purpose and format.

# Meta

## License

All information published in this document is hereby released in the public domain.

## Special Thanks

Thanks to the folks from #invidious on libera.chat for their invaluable assistance and insight.

Additional thanks to the maintainers of Invidious, Piped, youtube-dl, yt-dlp, uBlock Origin, and Sponsorblock for all their hard work in making YouTube a far more tolerable platform to interact with.

