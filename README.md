# SRS(Simple Realtime Server)


cd srs/trunk
./configure --help

chmod +x configure
chmod -R +x auto/
chmod -R +x 3rdparty/
sudo ./configure --gb28181=on

sudo make

sudo vim ./conf/push.gb28181.conf

```
# push gb28181 stream to SRS.

listen                  1935;
max_connections         1000;
# 注意：修改off为on，后台启动
daemon              on;
# 注意：注释此处 
#srs_log_tank        console;

http_api {
    enabled         on;
    listen          1985;
}

http_server {
    enabled         on;
    listen          8080;
}

stats {
    network         0;
}

stream_caster {
    enabled             on;
    caster              gb28181;

    # 转发流到rtmp服务器地址与端口
    # TODO: https://github.com/ossrs/srs/pull/1679/files#r400875104
    # [stream] is VideoChannelCodecID(视频通道编码ID) for sip
    # 自动创建的道通[stream] 是‘chid[ssrc]’ [ssrc]是rtp的ssrc
    # [ssrc] rtp中的ssrc
    output              rtmp://127.0.0.1:1935/live/[stream];
    
    # 接收设备端rtp流的多路复用端口
    listen              9000;
    # 多路复用端口类型，on为tcp，off为udp
    # 默认：on
    # 注意：修改此处为off，使用udp协议
    tcp_enable            off;

    # rtp接收监听端口范围，最小值
    rtp_port_min        58200;
    # rtp接收监听端口范围，最大值
    rtp_port_max        58300;

    # 是否等待关键帧之后，再转发，
    # off:不需等待，直接转发
    # on:等第一个关键帧后，再转发
    wait_keyframe       on;
    
    # rtp包空闲等待时间，如果指定时间没有收到任何包
    # rtp监听连接自动停止，发送BYE命令
    rtp_idle_timeout    30;

    # 是否转发音频流
    # 目前只支持aac格式，所以需要设备支持aac格式
    # on:转发音频
    # off:不转发音频，只有视频
    # *注意*!!!:flv 只支持11025  22050  44100 三种
    # 如果设备端没有三种中任何一个，转发时为自动选择一种格式
    # 同时也会将adts的头封装在flv aac raw数据中
    # 这样的话播放器为自动通过adts头自动选择采样频率
    # 像ffplay, vlc都可以，但是flash是没有声音，
    # 因为flash,只支持11025 22050 44100
    # 注意：根据需要开打音频
    audio_enable        off;


    # 服务器主机号，可以域名或ip地址
    # 也就是设备端将媒体发送的地址，如果是服务器是内外网
    # 需要写外网地址，
    # 调用api创建stream session时返回ip地址也是host
    # $CANDIDATE 是系统环境变量，从环境变量获取地址，如果没有配置，用*
    # *代表指定stats network 的网卡号地址，如果没有配置network，默认则是第0号网卡地址
    # TODO: https://github.com/ossrs/srs/pull/1679/files#r400917594
    host       $CANDIDATE;

    #根据收到ps rtp包自带创建rtmp媒体通道，不需要api接口创建
    #rtmp地址参数[stream] 就是通道id  格式chid[ssrc]
    #注意：打开此处，我们使用自己的sip信令服务器，srs收到rtp流后创建媒体通道即可
    auto_create_channel   on;

    sip {
        # 是否启用srs内部sip信令
        # 为on信令走srs, off 只转发ps流
        # 注意，关闭sip服务，我们使用自己的sip服务，涉及到业务处理
        enabled off;
        
        # sip监听udp端口
        listen              5060;
        
        # SIP server ID(SIP服务器ID).
        # 设备端配置编号需要与该值一致，否则无法注册
        serial              34020000002000000001;

        # SIP server domain(SIP服务器域)
        realm               3402000000;

        # 服务端发送ack后，接收回应的超时时间，单位为秒
        # 如果指定时间没有回应，认为失败
        ack_timeout         30;

        # 设备心跳维持时间，如果指定时间内(秒）没有接收一个心跳
        # 认为设备离线
        keepalive_timeout   120;

        # 注册之后是否自动给设备端发送invite
        # on: 是  off 不是，需要通过api控制
        auto_play           on;
        # 设备将流发送的端口，是否固定
        # on 发送流到多路复用端口 如9000
        # off 自动从rtp_mix_port - rtp_max_port 之间的值中
        # 选一个可以用的端口
        invite_port_fixed     on;

        # 向设备或下级域查询设备列表的间隔，单位(秒)
        # 默认60秒
        query_catalog_interval  60;
    }
}

rtc_server {
    enabled         on;
    # Listen at udp://8000
    listen          8000;
    #
    # The $CANDIDATE means fetch from env, if not configed, use * as default.
    #
    # The * means retrieving server IP automatically, from all network interfaces,
    # @see https://github.com/ossrs/srs/issues/307#issuecomment-599028124
    # 注意修改为流服务器地址
    # candidate       $CANDIDATE;
    candidate       10.10.2.213;
}

vhost __defaultVhost__ {
    rtc {
        enabled     on;
        rtmp_to_rtc on;
        bframe      discard;
    }

    http_remux {
        enabled     on;
        mount       [vhost]/[app]/[stream].flv;
    }
}
```

chmod +x ./etc/init.d/srs
sudo ./objs/srs -c ./conf/push.gb28181.conf

[2022-04-30 00:26:27.038][Trace][89679][925s1h86] XCORE-SRS/5.0.26(Leo)
[2022-04-30 00:26:27.038][Trace][89679][925s1h86] config parse complete
[2022-04-30 00:26:27.038][Warn][89679][925s1h86][22] transform: vhost.rtc.bframe to vhost.rtc.keep_bframe off
[2022-04-30 00:26:27.038][Trace][89679][925s1h86] you can check log by: tail -n 30 -f ./objs/srs.log
[2022-04-30 00:26:27.038][Trace][89679][925s1h86] please check SRS by: ./etc/init.d/srs status

进入srs控制台
http://10.10.2.29:8080/


![](http://ossrs.net/gif/v1/sls.gif?site=github.com&path=/srs/develop)
[![](https://github.com/ossrs/srs/actions/workflows/codeql-analysis.yml/badge.svg?branch=develop)](https://github.com/ossrs/srs/actions?query=workflow%3ACodeQL+branch%3Adevelop)
[![](https://github.com/ossrs/srs/actions/workflows/release.yml/badge.svg)](https://github.com/ossrs/srs/actions/workflows/release.yml?query=workflow%3ARelease)
[![](https://github.com/ossrs/srs/actions/workflows/test.yml/badge.svg?branch=develop)](https://github.com/ossrs/srs/actions?query=workflow%3ATest+branch%3Adevelop)
[![](https://codecov.io/gh/ossrs/srs/branch/develop/graph/badge.svg)](https://codecov.io/gh/ossrs/srs/branch/develop)
[![](https://ossrs.net/wiki/images/wechat-badge4.svg)](../../wikis/Contact#wechat)
[![](https://img.shields.io/twitter/follow/srs_server?style=social)](https://twitter.com/srs_server)
[![](https://badgen.net/discord/members/yZ4BnPmHAd)](https://discord.gg/yZ4BnPmHAd)
[![](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fossrs%2Fsrs.svg?type=small)](https://app.fossa.com/projects/git%2Bgithub.com%2Fossrs%2Fsrs?ref=badge_small)
[![](https://ossrs.net/wiki/images/srs-faq.svg)](https://github.com/ossrs/srs/issues/2716)
[![](https://badgen.net/badge/srs/stackoverflow/orange?icon=terminal)](https://stackoverflow.com/questions/tagged/simple-realtime-server)
[![](https://opencollective.com/srs-server/tiers/badge.svg)](https://opencollective.com/srs-server/contribute)
[![](https://ossrs.net/wiki/images/mulan-incubating.svg)](http://mulanos.cn)
[![](https://img.shields.io/youtube/channel/views/UCP6ZblCL_fIJoEyUzZxC1ng?style=social)](https://www.youtube.com/channel/UCP6ZblCL_fIJoEyUzZxC1ng)
[![](https://ossrs.net/wiki/images/srs-alternativeto.svg)](https://alternativeto.net/software/srs/about/)
[![](https://img.shields.io/docker/pulls/ossrs/srs)](https://hub.docker.com/r/ossrs/srs/tags)

SRS/5.0 is a simple, high efficiency and realtime video server, supports RTMP/WebRTC/HLS/HTTP-FLV/SRT/GB28181.

SRS/5.0，[Bee](https://github.com/ossrs/srs/wiki/Product#release50) 是一个简单高效的实时视频服务器，支持RTMP/WebRTC/HLS/HTTP-FLV/SRT/GB28181。

SRS is licenced under [MIT](https://github.com/ossrs/srs/blob/develop/LICENSE) or [MulanPSL-2.0](https://spdx.org/licenses/MulanPSL-2.0.html),
and note that [MulanPSL-2.0 is compatible with Apache-2.0](https://www.apache.org/legal/resolved.html#category-a),
but some third-party libraries are distributed using their [own licenses](https://github.com/ossrs/srs/wiki/LicenseMixing).

[![](https://mermaid.ink/img/pako:eNqNkkFP4zAQhf-K5TPBHOBSEBIkrXIItIqrckg4uPEktZTY1dgpWyH--9ppKsIuSMzBivXeG3_25J1WRgKd0bo1b9VOoCNZfltq4sv22wbFfkfQVcULbPN1TOJWgXb29eQIJRVC5ZTRn8FQ6U0R79B0wBbeUJs_d1tk92wuG2Bc1AIVAVe9ThLPwqkDFGrJ2YOWaJQ8q6DlP0RtcGZ-IdwhiE7p5vdooMOdsVgsuj00bPnIyQBH5ifBTqn2rTh6b3rDNlnM6jrsR_tqkOzPlAZVo3TBc06uL6_I8gB4UPA2xrmHboE8-yPJt9DrxwmHRfvZaOh7buObAv4PgZZE0WlqURSFGd5OlXz9tGI8XwdtfJAveppxtsg2bOK7Hx_jiy9uTS-5MygaGIBCMHngKUsT_lNoYxLCj9ZBN0SSTU7CWU-r62mCXtAOsBNK-v_zPeRL6nbQQUln_lNCLfrWlbTUH97a76VwMJfKs9BZLVoLF1T0zvCjrujMYQ9nU6KEH1A3uj7-Ah_g8P8)](https://mermaid.live/edit#pako:eNqNkkFP4zAQhf-K5TPBHOBSEBIkrXIItIqrckg4uPEktZTY1dgpWyH--9ppKsIuSMzBivXeG3_25J1WRgKd0bo1b9VOoCNZfltq4sv22wbFfkfQVcULbPN1TOJWgXb29eQIJRVC5ZTRn8FQ6U0R79B0wBbeUJs_d1tk92wuG2Bc1AIVAVe9ThLPwqkDFGrJ2YOWaJQ8q6DlP0RtcGZ-IdwhiE7p5vdooMOdsVgsuj00bPnIyQBH5ifBTqn2rTh6b3rDNlnM6jrsR_tqkOzPlAZVo3TBc06uL6_I8gB4UPA2xrmHboE8-yPJt9DrxwmHRfvZaOh7buObAv4PgZZE0WlqURSFGd5OlXz9tGI8XwdtfJAveppxtsg2bOK7Hx_jiy9uTS-5MygaGIBCMHngKUsT_lNoYxLCj9ZBN0SSTU7CWU-r62mCXtAOsBNK-v_zPeRL6nbQQUln_lNCLfrWlbTUH97a76VwMJfKs9BZLVoLF1T0zvCjrujMYQ9nU6KEH1A3uj7-Ah_g8P8)

> Note: The single node architecture for SRS, generally and major use scenario.

[![SRS Overview](https://ossrs.net/wiki/images/SRS-Overview-4.0.png)](https://ossrs.net/wiki/images/SRS-Overview-4.0.png)

> Note: The cluster architecture for SRS. If image load fail, please see it at [here](https://www.processon.com/view/link/619f29791efad425fd699fd2).

<a name="product"></a>
<a name="usage-docker"></a>
## Usage

Build SRS from source:

```
git clone -b develop https://gitee.com/ossrs/srs.git &&
cd srs/trunk && ./configure && make && ./objs/srs -c conf/srs.conf
```

Open [http://localhost:8080/](http://localhost:8080/) to check it, then publish
by [FFmpeg](https://ffmpeg.org/download.html) or [OBS](https://obsproject.com/download) as:

```bash
ffmpeg -re -i ./doc/source.flv -c copy -f flv -y rtmp://localhost/live/livestream
```

Play the following streams by [players](https://ossrs.net):

* RTMP (by [VLC](https://www.videolan.org/)): rtmp://localhost/live/livestream
* H5(HTTP-FLV): [http://localhost:8080/live/livestream.flv](http://localhost:8080/players/srs_player.html?autostart=true&stream=livestream.flv&port=8080&schema=http)
* H5(HLS): [http://localhost:8080/live/livestream.m3u8](http://localhost:8080/players/srs_player.html?autostart=true&stream=livestream.m3u8&port=8080&schema=http)

Note that if convert RTMP to WebRTC, please use [`rtmp2rtc.conf`](https://github.com/ossrs/srs/issues/2728#issuecomment-964686152):

* H5(WebRTC): [webrtc://localhost/live/livestream](http://localhost:8080/players/rtc_player.html?autostart=true)

> Note: Besides of FFmpeg or OBS, it's also able to [publish by H5](http://localhost:8080/players/rtc_publisher.html?autostart=true) 
> if **WebRTC([CN](https://github.com/ossrs/srs/wiki/v4_CN_WebRTC#rtc-to-rtmp), [EN](https://github.com/ossrs/srs/wiki/v4_EN_WebRTC#rtc-to-rtmp))** is enabled,
> please remember to set the **CANDIDATE([CN](https://github.com/ossrs/srs/wiki/v4_CN_WebRTC#config-candidate) or [EN](https://github.com/ossrs/srs/wiki/v4_EN_WebRTC#config-candidate))** for WebRTC.

> Highly recommend that directly run SRS by
> **docker([CN](https://github.com/ossrs/srs/wiki/v4_CN_Home#docker) / [EN](https://github.com/ossrs/srs/wiki/v4_EN_Home#docker))**,
> **Cloud Virtual Machine([CN](https://github.com/ossrs/srs/wiki/v4_CN_Home#cloud-virtual-machine) / [EN](https://github.com/ossrs/srs/wiki/v4_EN_Home#cloud-virtual-machine))**,
> or **K8s([CN](https://github.com/ossrs/srs/wiki/v4_CN_Home#k8s) / [EN](https://github.com/ossrs/srs/wiki/v4_EN_Home#k8s))**,
> however it's also easy to build SRS from source code, for detail please see
> **Getting Started([CN](https://github.com/ossrs/srs/wiki/v4_CN_Home#getting-started) / [EN](https://github.com/ossrs/srs/wiki/v4_EN_Home#getting-started))**.

> Note: If need HTTPS, by which WebRTC and modern browsers require, please read
> **HTTPS API([CN](https://github.com/ossrs/srs/wiki/v4_CN_HTTPApi#https-api) / [EN](https://github.com/ossrs/srs/wiki/v4_EN_HTTPApi#https-api))**
> and **HTTPS Callback([CN](https://github.com/ossrs/srs/wiki/v4_CN_HTTPCallback#https-callback) / [EN](https://github.com/ossrs/srs/wiki/v4_EN_HTTPCallback#https-callback))**
> and **HTTPS Live Streaming([CN](https://github.com/ossrs/srs/wiki/v4_EN_DeliveryHttpStream#https-flv-live-stream) / [EN](https://github.com/ossrs/srs/wiki/v4_EN_DeliveryHttpStream#https-flv-live-stream))**,
> however HTTPS proxy also works perfect with SRS such as Nginx.

<a name="srs-40-wiki"></a>
<a name="wiki"></a>

From here, please read wikis:

* [Getting Started](https://github.com/ossrs/srs/wiki/v4_EN_Home#getting-started), please read Wiki first.
* [中文文档：起步](https://github.com/ossrs/srs/wiki/v4_CN_Home#getting-started)，不读Wiki一定扑街，不读文档请不要提Issue，不读文档请不要提问题，任何文档中明确说过的疑问都不会解答。

Fast index for Wikis:

* Overview? ([CN](https://github.com/ossrs/srs/wiki/v4_CN_Home), [EN](https://github.com/ossrs/srs/wiki/v4_EN_Home))
* How to deliver RTMP streaming?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleRTMP), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleRTMP))
* How to build RTMP Edge-Cluster?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleRTMPCluster), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleRTMPCluster))
* How to build RTMP Origin-Cluster?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleOriginCluster), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleOriginCluster))
* How to deliver HTTP-FLV streaming?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleHttpFlv), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleHttpFlv))
* How to deliver HLS streaming?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleHLS), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleHLS))
* How to deliver low-latency streaming?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleRealtime), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleRealtime))
* How to use WebRTC? ([CN](https://github.com/ossrs/srs/wiki/v4_CN_WebRTC), [EN](https://github.com/ossrs/srs/wiki/v4_EN_WebRTC))

Other important wiki:

* Usage: How to publish GB28181 to SRS? [#1500](https://github.com/ossrs/srs/issues/1500#issuecomment-606695679)
* Usage: How to delivery DASH(Experimental)?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleDASH), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleDASH))
* Usage: How to transode RTMP stream by FFMPEG?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleFFMPEG), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleFFMPEG))
* Usage: How to delivery HTTP FLV Live Streaming Cluster?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleHttpFlvCluster), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleHttpFlvCluster))
* Usage: How to ingest file/stream/device to RTMP?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleIngest), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleIngest))
* Usage: How to forward stream to other servers?([CN](https://github.com/ossrs/srs/wiki/v4_CN_SampleForward), [EN](https://github.com/ossrs/srs/wiki/v4_EN_SampleForward))
* Usage: How to improve edge performance for multiple CPUs? ([CN](https://github.com/ossrs/srs/wiki/v4_CN_REUSEPORT), [EN](https://github.com/ossrs/srs/wiki/v4_EN_REUSEPORT))
* Usage: How to file a bug or contact us? ([CN](https://github.com/ossrs/srs/wiki/v4_CN_Contact), [EN](https://github.com/ossrs/srs/wiki/v4_EN_Contact))

## AUTHORS

Thank you to all our contributors! 🙏

[![](https://opencollective.com/srs-server/contributors.svg?width=800&button=false)](https://opencollective.com/srs-server/contribute)

> Note: You may provide financial support for this project by donating [via Open Collective](https://opencollective.com/srs-server/contribute). Thank you for your support!

The [TOC(Technical Oversight Committee)](trunk/AUTHORS.md#toc), [Developers](trunk/AUTHORS.md#developers) and [contributors](trunk/AUTHORS.md#contributors):

* [Winlin](https://github.com/winlinvip): Focus on [ST](https://github.com/ossrs/state-threads) and [Issues/PR](https://github.com/ossrs/srs/issues).
* [ZhaoWenjie](https://github.com/wenjiegit): Focus on [HDS](https://github.com/simple-rtmp-server/srs/wiki/v4_CN_DeliveryHDS) and [Windows](https://github.com/ossrs/srs/issues/2532).
* [ShiWei](https://github.com/runner365): Focus on [SRT](https://github.com/simple-rtmp-server/srs/wiki/v4_CN_SRTWiki) and [H.265](https://github.com/ossrs/srs/issues/465).
* [XiaoZhihong](https://github.com/xiaozhihong): Focus on [WebRTC/QUIC](https://github.com/ossrs/srs/issues/2091) and [SRT](https://github.com/simple-rtmp-server/srs/wiki/v4_CN_SRTWiki).
* [WuPengqiang](https://github.com/Bepartofyou): Focus on [H.265](https://github.com/ossrs/srs/issues/465).
* [XiaLixin](https://github.com/xialixin): Focus on [GB28181](https://github.com/ossrs/srs/issues/1500).
* [LiPeng](https://github.com/lipeng19811218): Focus on [WebRTC](https://github.com/simple-rtmp-server/srs/wiki/v4_CN_WebRTC).
* [ChenGuanghua](https://github.com/chen-guanghua): Focus on [WebRTC/QoS](https://github.com/ossrs/srs/issues/2051).
* [ChenHaibo](https://github.com/duiniuluantanqin): Focus on [GB28181](https://github.com/ossrs/srs/issues/1500) and [API](https://github.com/ossrs/srs/issues/1657).

A big `THANK YOU` also goes to:

* All [contributors](trunk/AUTHORS.md#contributors) of SRS.
* All friends of SRS for [big supports](https://github.com/ossrs/srs/wiki/Product).
* [Genes](http://sourceforge.net/users/genes), [Mabbott](http://sourceforge.net/users/mabbott) and [Michael Talyanksy](https://github.com/michaeltalyansky) for creating and introducing [st](https://github.com/ossrs/state-threads/tree/srs).

## Contributing

We are grateful to the community for contributing bugfix and improvements, please follow the
[guide](https://github.com/ossrs/srs/contribute).

## LICENSE

[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fossrs%2Fsrs.svg?type=small)](https://app.fossa.com/projects/git%2Bgithub.com%2Fossrs%2Fsrs?ref=badge_small)

SRS is licenced under [MIT](https://github.com/ossrs/srs/blob/develop/LICENSE) or [MulanPSL-2.0](https://spdx.org/licenses/MulanPSL-2.0.html),
and note that [MulanPSL-2.0 is compatible with Apache-2.0](https://www.apache.org/legal/resolved.html#category-a),
but some third-party libraries are distributed using their [own licenses](https://github.com/ossrs/srs/wiki/LicenseMixing).

[![](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fossrs%2Fsrs.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Fossrs%2Fsrs?ref=badge_large)

## Releases

* 2022-03-19, Release [v4.0-b10](https://github.com/ossrs/srs/releases/tag/v4.0-b10), v4.0-b10, 4.0 beta10, v4.0.251, 144665 lines.
* 2022-02-15, Release [v4.0-b9](https://github.com/ossrs/srs/releases/tag/v4.0-b9), v4.0-b9, 4.0 beta9, v4.0.245, 144474 lines.
* 2022-02-11, Release [v4.0-b8](https://github.com/ossrs/srs/releases/tag/v4.0-b8), v4.0-b8, 4.0 beta8, v4.0.241, 144445 lines.
* 2022-02-09, Release [v4.0-b7](https://github.com/ossrs/srs/releases/tag/v4.0-b7), v4.0-b7, 4.0 beta7, v4.0.240, 144437 lines.
* 2022-02-04, Release [v4.0-b6](https://github.com/ossrs/srs/releases/tag/v4.0-b6), v4.0-b6, 4.0 beta6, v4.0.238, 144437 lines.
* 2022-01-30, Release [v4.0-b5](https://github.com/ossrs/srs/releases/tag/v4.0-b5), v4.0-b5, 4.0 beta5, v4.0.236, 144416 lines.
* 2022-01-17, Release [v4.0-b4](https://github.com/ossrs/srs/releases/tag/v4.0-b4), v4.0-b4, 4.0 beta4, v4.0.230, 144393 lines.
* 2022-01-13, Release [v4.0-b3](https://github.com/ossrs/srs/releases/tag/v4.0-b3), v4.0-b3, 4.0 beta3, v4.0.229, 144393 lines.
* 2022-01-03, Release [v4.0-b2](https://github.com/ossrs/srs/releases/tag/v4.0-b2), v4.0-b2, 4.0 beta2, v4.0.215, 144278 lines.
* 2021-12-19, Release [v4.0-b1](https://github.com/ossrs/srs/releases/tag/v4.0-b1), v4.0-b1, 4.0 beta1, v4.0.206, 144126 lines.
* 2021-12-01, Release [v4.0-b0](https://github.com/ossrs/srs/releases/tag/v4.0-b0), v4.0-b0, 4.0 beta0, v4.0.201, 144022 lines.
* 2021-11-15, Release [v4.0.198](https://github.com/ossrs/srs/releases/tag/v4.0.198), 4.0 dev8, v4.0.198, 144010 lines.
* 2021-11-02, Release [v4.0.191](https://github.com/ossrs/srs/releases/tag/v4.0.191), 4.0 dev7, v4.0.191, 143890 lines.
* 2021-10-12, Release [v4.0.177](https://github.com/ossrs/srs/releases/tag/v4.0.177), 4.0 dev6, v4.0.177, 143686 lines.
* 2021-09-05, Release [v4.0.161](https://github.com/ossrs/srs/releases/tag/v4.0.161), 4.0 dev5, v4.0.161, 145865 lines.
* 2021-08-15, Release [v4.0.156](https://github.com/ossrs/srs/releases/tag/v4.0.156), 4.0 dev4, v4.0.156, 145490 lines.
* 2021-08-14, Release [v4.0.153](https://github.com/ossrs/srs/releases/tag/v4.0.153), 4.0 dev3, v4.0.153, 145506 lines.
* 2021-08-07, Release [v4.0.150](https://github.com/ossrs/srs/releases/tag/v4.0.150), 4.0 dev2, v4.0.150, 145289 lines.
* 2021-07-25, Release [v4.0.146](https://github.com/ossrs/srs/releases/tag/v4.0.146), 4.0 dev1, v4.0.146, 144026 lines.
* 2021-07-04, Release [v4.0.139](https://github.com/ossrs/srs/releases/tag/v4.0.139), 4.0 dev0, v4.0.139, 143245 lines.
* 2020-06-27, [Release v3.0-r0](https://github.com/ossrs/srs/releases/tag/v3.0-r0), 3.0 release0, 3.0.141, 122674 lines.
* 2020-02-02, [Release v3.0-b0](https://github.com/ossrs/srs/releases/tag/v3.0-b0), 3.0 beta0, 3.0.112, 121709 lines.
* 2019-10-04, [Release v3.0-a0](https://github.com/ossrs/srs/releases/tag/v3.0-a0), 3.0 alpha0, 3.0.56, 107946 lines.
* 2017-03-03, [Release v2.0-r0](https://github.com/ossrs/srs/releases/tag/v2.0-r0), 2.0 release0, 2.0.234, 86373 lines.
* 2016-08-06, [Release v2.0-b0](https://github.com/ossrs/srs/releases/tag/v2.0-b0), 2.0 beta0, 2.0.210, 89704 lines.
* 2015-08-23, [Release v2.0-a0](https://github.com/ossrs/srs/releases/tag/v2.0-a0), 2.0 alpha0, 2.0.185, 89022 lines.
* 2014-12-05, [Release v1.0-r0](https://github.com/ossrs/srs/releases/tag/v1.0-r0), all bug fixed, 1.0.10, 59391 lines.
* 2014-10-09, [Release v0.9.8](https://github.com/ossrs/srs/releases/tag/v0.9.8), all bug fixed, 1.0.0, 59316 lines.
* 2014-04-07, [Release v0.9.1](https://github.com/ossrs/srs/releases/tag/v0.9.1), live streaming. 30000 lines.
* 2013-10-23, [Release v0.1.0](https://github.com/ossrs/srs/releases/tag/v0.1.0), rtmp. 8287 lines.
* 2013-10-17, Created.

## Features

Please read [FEATURES](trunk/doc/Features.md#features).

<a name="history"></a>
<a name="change-logs"></a>

## Changelog

Please read [CHANGELOG](trunk/doc/CHANGELOG.md#changelog).

## Compare

Comparing with other media servers, SRS is much better and stronger, for details please 
read Product([CN](https://github.com/ossrs/srs/wiki/v4_CN_Compare)/[EN](https://github.com/ossrs/srs/wiki/v4_EN_Compare)).

## Performance

Please read [PERFORMANCE](trunk/doc/PERFORMANCE.md#performance).

## Architecture

Please read [ARCHITECTURE](trunk/doc/Architecture.md#architecture).

## Ports

Please read [PORTS](trunk/doc/Resources.md#ports).

## APIs

Please read [APIS](trunk/doc/Resources.md#apis).

## Mirrors

Please read [MIRRORS](trunk/doc/Resources.md#mirrors).

Beijing, 2013.10<br/>
Winlin

