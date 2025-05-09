# Wowza bug report: extended timecode handling

This repository contains a minimal setup to reproduce a (suspected) bug in Wowza Streaming Engine concerning RTMP extended timecode handling causing streams to be unplayable (A/V sync issues).

## Summary

The bug occurs when streaming from a Kiloview E3 encoder to Wowza Streaming Engine using the RTMP protocol. When the encoder sends an extended timecode (very early in the stream) it appears that Wowza handles it incorrectly. The result is a stream that cannot be played/recorded properly.

## Environment

### Kiloview E3

Kiloview E3 hardware version is 4.0, firmware version is 1.10.0005. This information can be found via the web interface of the encoder under _Settings > System Settings > Firmware update_.

The Kiloview E3 encoder is advertising itself as "GStreamer 1.22.3 FLV muxer". From Kiloview logging it looks like it is using the [`rtmp2sink`](https://gstreamer.freedesktop.org/documentation/rtmp2/rtmp2sink.html?gi-language=c) element in the Gstreamer pipeline.

### Wowza Streaming Engine 4.9.3

The version of Wowza used in this report is 4.9.3 (Docker `image: wowzamedia/wowza-streaming-engine-linux:4.9.3`) but the same problem occurs with previous versions of Wowza Streaming Engine (at least version 4.7.7 also encounters this problem).

#### Configuration: `live/Application.xml`

The default configuration of Wowza Streaming Engine is used (from the Docker image), with the following changes (see commits).

- Change `<Streams><StreamType>` to `live-record` so all incoming streams are recorded.
- Add to `<Streams><Properties>` the following property:

  ```xml
    <Property>
     <Name>debugIncomingPacketTimes</Name>
     <Value>true</Value>
     <Type>Boolean</Type>
    </Property>
  ```

### Wireshark

Use Wireshark to capture the network traffic on the interface where Wowza is running. Use the following filter to only capture RTMP packets: `rtmpt`.

_Note: You can also use `tcpdump` to capture the packets, but Wireshark is more user-friendly._

## Steps to reproduce

The bug occurs immediately after starting a stream from a Kiloview E3 encoder to Wowza Streaming Engine.

1. Setup HDMI input on Kiloview E3 encoder (any source should do).

2. Add a new "Stream Service" of type RTMP via the webinterface for Kiloview E3. Indicate that the stream should be recorded in FLV format (hence the `flv:` prefix).

    ```plain
    rtmp://wowza:wowza@`ifconfig.me`/live/flv:test
    ```

3. Start Wowza by running `docker compose up` in the root of this repository.

4. Start capturing network traffic using Wireshark.

5. Start the stream on the Kiloview E3 encoder.

6. Try to play the RTMP stream via VLC or `ffplay` (e.g. `ffplay rtmp://localhost/live/test`). Or try to play the HLS stream via VLC or `ffplay` (e.g. `ffplay http://localhost:1935/live/test/playlist.m3u8`).

7. If the stream is not playable, secure the logs, FLV recording and network packet capture.

    _Note: Sporadically the stream does not contain extended timecodes, in which case the stream can be played normally. In that case, stop and start the stream until you see the problem._

8. You will see many messages like the following generated showing that the `debugIncomingPacketTimes` property is in effect.

    ```plain
    INFO server comment - LiveReceiver.audio[live/_definst_/test]: hdr:0xaf sz:00361 tc:4294967715/23 rt:1746641073827/0
    ```

## Expected result

The stream and the `content/test.flv` should be playable via VLC or `ffplay` without any A/V sync issues.

## Actual result

The stream is not playable via VLC or `ffplay`. The recording `content/test.flv` is also not playable.

## Proof

- `proof/WowzaStreamingEngine.log`
  - The Wowza log file copied from `supervisor/WowzaStreamingEngine.log`.
- `proof/test.flv` (copied from `content/test.flv`)
  - The (broken) recording. It appears to play fine using `ffplay` but the video freezes when played in VLC.
- `proof/packetcapture-rtmpt.pcapng.gz`
  - The captured RTMP network traffic which can be loaded into Wireshark (preferably) or previewed in the terminal with `tshark -r proof/packetcapture-rtmpt.pcapng.gz`.

## Severity / Priority

The bug is critical as it prevents the use of the Kiloview E3 encoder with Wowza Streaming Engine. The bug occurs immediately after starting a stream from the Kiloview E3 encoder to Wowza Streaming Engine.
