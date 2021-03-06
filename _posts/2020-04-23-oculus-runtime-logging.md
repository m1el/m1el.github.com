---
layout: post
permalink: /oculus-tls-extract/results.html
title: What does Oculus Runtime send home?
date: 2020-04-23 13:17
---

I wanted to know what data Oculus Runtime sends home.
To do that, I've [implemented a custom tool](/oculus-tls-extract/)
to extract TLS keys from a running Oculus Runtime instance.

You can find my findings below.

<!-- more -->

## Methodology

I've captured HTTP packets using Wireshark, and decrypted them using SSL KEYLOG generated by [oculus-tls-extractor][oculus-tls-extractor].

Then I've manually inspected the captured packets over multiple sessions.

## Limitations

- I was not able to inspect data from OculusClient, which is an Electron application, and there may be additional data being sent from there.
- I manually inspected the packets and it is certain that I've missed something.

## Findings

The most used endpoints were: `https://graph.oculus.com/logging_client_events` and `https://graph.oculus.com/graphql`.

`/graphql` endpoint was mostly used as a read-only endpoint, with few exceptions.

For example:

```
Request:
POST /graphql?forced_locale=en_US HTTP/1.1
Host: graph.oculus.com
...

access_token=<removed_access_token>&variables=%7B%7D&doc_id=<removed_doc_id>

---
Response:
HTTP/1.1 200 OK
...
Access-Control-Allow-Origin: https://facebook.com
Content-Type: text/html; charset="utf-8"
...

{"data":{"viewer":{"user":{"current_party":null,"friends":{"count":0,"edges":[]},"id":"<my user id>"}}}}
```

There were a few exceptions.

```
POST /graphql

Form data:
method = POST
forced_locale = en_US
doc_id = <removed_doc_id>
variables = {
  "input": {
    "client_mutation_id": "30166",
    "device_id": "<device uuid>",
    "device_name": "UNGNU",
    "device_serial": "<another uuid>",
    "hardware": [
      {
        "battery": "DISCHARGING",
        "battery_percent": 0,
        "connection": "connected",
        "operational": "operable",
        "serial": "<device serial>",
        "type": "HMD_LAGUNA"
      },
      {
        "battery": "DISCHARGING",
        "battery_percent": 0,
        "connection": "disconnected",
        "operational": "inoperable",
        "serial": "<device serial>",
        "type": "INPUT_LLCON"
      },
      {
        "battery": "DISCHARGING",
        "battery_percent": 0,
        "connection": "disconnected",
        "operational": "inoperable",
        "serial": "<device serial>",
        "type": "INPUT_RLCON"
      }
    ],
    "meets_min_spec": "PASS"
  }
}
```


`/logging_client_events` this is a more interesting endpoint.
It has a stream of data being sent by the runtime.

The requests have the following format:

```
Request:
POST /logging_client_events HTTP/1.1
Host: graph.oculus.com
Content-Type: application/json
...

{
  "message": {
    "app_id": 490451621120355,
    "app_ver": "16.0.0.118.452",
    "data": [
      {
        "extra": {
          "client_session_id": 1,
          "hmd_firmware_version": "2.2.0",
          "hmd_os_build": "0",
          "hmd_type": "HMD KM",
          "id": "<user_id>",
          "in_staged_rollout": false,
          "is_core2": true,
          "is_dash_up": false,
          "left_lcon_firmware_version": "unavailable",
          "release_channel": "LIVE",
          "remote_firmware_version": "unavailable",
          "right_lcon_firmware_version": "unavailable",
          "timestamp_gaps": "8",
          "tracker_firmware_version": "",
          "valid_imu_sample": "57937"
        },
        "name": "hal_imu",
        "time": 1587645768.38274
      }
    ],
    "device_id": "<device_id>",
    "log_type": "client_event",
    "oculus_access_token": "<access_token>",
    "oculus_userid": "<user_id>",
    "seq": 3,
    "session_id": "1587645706.843546",
    "time": 1587645777.5674851,
    "uid": "0"
  }
}

Response:
HTTP/1.1 200 OK

{"checksum":"","config":"","app_data":"{}"}
```

This is a remote logging endpoint.  I've dumped all logs, sorted them by types and inspected a few log entries of each type.

If you want to inspect the data yourself, I've uploaded a my (cleaned up) log entries in LJSON format [here][some-log].

One of the events caught my eye:

```
{
  "extra": {
    ...
    "level": "Error",
    "message": "c:\\cygwin\\data\\sandcastle\\boxes\\trunk-hg-ovrsource-win
     \\software\\oculussdk\\pc\\oaf\\liboaf\\steamparser\\parsing.cpp(163)
     : Error parsing Steam info (1971039)",
    ...
  },
  "name": "oculus_application_framework_error",
  "time": 1587238393.064477
}
```

It looks like Oculus Runtime tries to parse Steam information, but fails to do so?  I would appreciate if it didn't.


OK, let's look at `oculus_service_start` entry: https://gist.github.com/m1el/373b4fa14a46b22f933b44d96dd26f7b

```
      "CpuList": [
        {
...
          "MaxFrequencyMhz": 3600,
          "Model": 0,
          "Name": "Intel(R) Core(TM) i9-9900K CPU @ 3.60GHz",
          "NumberOfCores": 8,
          "NumberOfLogicalProcessors": 16,
...
      "UsbMatchedController": {
        "Compatibility": 3,
        "Compatibility_str": "UNKNOWN_USB3",
        "DriverDescription": "USB xHCI Compliant Host Controller",
...
    "monitors": [
      { "horizontal_active_pixels": 3840, "horizontal_image_size_mm": 1872, "vertical_active_pixels": 2160, "vertical_image_size_mm": 1053 },
      { "horizontal_active_pixels": 1920, "horizontal_image_size_mm": 518, "vertical_active_pixels": 1200, "vertical_image_size_mm": 324 },
      { "horizontal_active_pixels": 1920, "horizontal_image_size_mm": 510, "vertical_active_pixels": 1080, "vertical_image_size_mm": 287 } ],
...
      "VideoCardList": [
        {
          "LUID": "<LUID>",
          "Name": "AMD Radeon R9 290/390",
...
    "motherboard_manufacturer": "ASUSTeK COMPUTER INC.",
    "motherboard_product": "PRIME Z390-A",
    "motherboard_serial": "<motherboard_serial>",
    "motherboard_version": "Rev 1.xx",
    "os": "Windows 10 64-bit",
    "os_build": "18363",
...
```

That's a lot of info about the system!  All GPUs, all monitors, USB controllers, motherboard serial number, etc.

The rest of the logging events were rather boring.  Most of them were:
- applications being launched in VR
- guardian area
- debug information about inside-out tracking
- lots and lots of debug information metrics
- some cryptic errors
- no information about room geometry or photos of the room

## Conclusions

The findings were rather boring.  The most egregious things I've found were:

1. Sending a dump of system information on service startup.
2. Trying to parse Steam information?

I would like it if Oculus didn't send this information home, but this probably won't happen.

My analysis of the logs is incomplete, if you'd like to do inspect the data yourself, feel free to
take a look at [the logs I've captured myself][some-log] or capture your own logs using [the tool I wrote][oculus-tls-extractor].

[some-log]: https://gist.github.com/m1el/0f10913c1a58ba1cea92a813065ab857 "Edited logs from oculus runtime"
[oculus-tls-extractor]: https://github.com/m1el/oculus-tls-extractor "Oculus TLS Extractor"
[hw-info]: https://gist.github.com/m1el/373b4fa14a46b22f933b44d96dd26f7b "Hardware dump"
