# Live Streaming Video on the Web using SRS

SRS Stack is an all-in-one, out-of-the-box, and open-source video solution for creating online video services, including live streaming and WebRTC, on the cloud or through self-hosting.

SRS Stack makes it easy for you to create an online video service. It is made using Go, Reactjs, SRS, FFmpeg, and WebRTC. It supports protocols like RTMP, WebRTC, HLS, HTTP-FLV, and SRT. It offers features like authentication, streaming on multiple platforms, recording, transcoding, virtual live events, automatic HTTPS, and an easy-to-use HTTP Open API.
### Notes

1. Video source must be `h264` encoded for max compatibility. Set the camera to encode h264 at source so transcoding can be avoided later.

2. HTTPS is required for publishing streams using WebRTC, and it improves security. If you want to support the video streaming in any HTTPS website, such as a WordPress website, you must use HLS/FLV/WebRTC with HTTPS, or it will fail for security reasons.

## Step 1: Docker Run
    
Run srs-stack in one docker, then open [http://localhost:2022](http://localhost:2022/) in browser:

```shell
docker run --restart always -d -it --name srs-stack -v $HOME/data:/data \
  -p 2022:2022 -p 2443:2443 -p 1935:1935 -p 8000:8000/udp -p 10080:10080/udp \
  ossrs/srs-stack:5
```

Important: To use WebRTC in a browser, avoid using localhost or 127.0.0.1. Instead, use a private IP (e.g., [https://192.168.3.85:2443](https://192.168.3.85:2443/)), a public IP (e.g., [https://136.12.117.13:2443](https://136.12.117.13:2443/)), or a domain (e.g., [https://your-domain.com:2443](https://your-domain.com:2443/)). To set up HTTPS, refer to [this post](https://blog.ossrs.io/how-to-secure-srs-with-lets-encrypt-by-1-click-cb618777639f).

## Step 1.2 Docker Compose

```yaml
# docker compose to srs-stack
# Filename: docker-compose.yaml
name: livevideo-example-com

version: '3'

services:
  livevideo:
    image: ossrs/srs-stack:5.13.13
    container_name: livevideo.example.com
    restart: always

    network_mode: host

    environment:
      - CANDIDATE="livevideo.example.com"

    volumes:
      - ./data:/data

    labels:
      - com.centurylinklabs.watchtower.enable=true
      - traefik.enable=true
      - traefik.http.routers.livevideo.rule=Host(`livevideo.example.com`)
      - traefik.http.routers.livevideo.tls=true
      - traefik.http.routers.livevideo.tls.certresolver=lets-encrypt
      - traefik.http.services.livevideo.loadbalancer.server.port=2022
```

## Step 1.3 Deploy SRS Stack in docker

Run `docker-compose up -d` to deploy the srs-stack.

After creating the SRS Stack, you can access it through `http://livevideo.example.com/mgmt` via a browser.

## Step 2: # Live Streaming Setup with FFmpeg and SRS

Open the SRS Stack, select `Scenarios > Streaming > RTMP: FFmpeg`, and copy the stream URL from the `RTMP: FFmpeg` usage section.

Example Publish Stream URL: `rtmp://livevideo.example.com/live/paemog?secret=743acd8e22af4bc9b5c358e63704e0ba

## Step 3: Setup ffmpeg

ffmpeg used for pulling a camera feed over rtsp and forwarding to WHEP player or ffplay with the correct streaming container format.

### Input From File for Testing

```bash
ffmpeg -re -i test.mp4 -c copy -f flv rtmp://livevideo.example.com/live/paemog?secret=743acd8e22af4bc9b5c358e63704e0ba
```
### Notes

1. `-re`: play back file in real time by reading input at native frame rate, eg 1 hour video will play back in 1 hour.
2. `-i test.mp4`: sample input file for streaming.
3. `-c copy`: don't transcode the audio/video stream, just copy to output.
4. `-f flv`: suitable streaming format.
5. `rtmp://livevideo.example.com/live/paemog?secret=743acd8e22af4bc9b5c358e63704e0ba`: srs rtmp server location for sending the output stream.

### Input From CCTV Camera

HikVision CCTV cameras provide RTSP stream in the default `tcp:554` port and no path is required. Username and password are the same as what is required to access the camera web portal. Ensure the camera is set for `h264` encoding.

```bash
# camera IP: c.c.c.c
ffmpeg -re -rtsp_transport tcp -i rtsp://username:password@c.c.c.c -flvflags no_duration_filesize -c copy -f flv rtmp://livevideo.example.com/live/paemog?secret=743acd8e22af4bc9b5c358e63704e0ba
```

`-rtsp_transport tcp`: Specifies the transport protocol to be used for the RTSP connection. In this case, it's set to TCP, which can be more reliable than UDP in certain network conditions.

`-flvflags no_duration_filesize`: This option is used to set FLV (Flash Video) flags. In this case, it's set to `no_duration_filesize`, which means FFmpeg will not write duration and filesize to the FLV header.
### Notes

1. If ffmpeg is segfaulting, use the ubuntu supplied ffmpeg binary (`apt install ffmpeg`) and not the static builds from ffmpeg.org.

## Step 4: Play WebRTC stream, WHEP player

After publishing the stream, you can view it with a WebRTC HTML5 player. 
Access the WHEP player from the `RTMP: FFmpeg` usage section.

Example: Play WebRTC stream, WHEP player: `https://livevideo.example.com/rtc/v1/whep/?app=live&stream=paemog`

## Step 5: Latency Check

For the `HikVision camera model DS-2CD3026G2-IS` we get 1.4 seconds latency , as observed with the following configuration.

The CPU utilization is `4%` across `4 CPUs`, and the `memory usage` is `0%` out of 34MB in a total of 16GB for 6 video streams.

![[video-camera-config.png]]

## Step 5.1: WebRTC stream, WHEP player results screenshots

![[video-streaming-result-1.png]]


![[video-streaming-result-2.png]]
## References
1. [FFmpeg Live Streaming Guide](https://sonnati.wordpress.com/2011/08/30/ffmpeg-%e2%80%93-the-swiss-army-knife-of-internet-streaming-%e2%80%93-part-iv/)
