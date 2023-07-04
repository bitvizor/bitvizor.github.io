---
layout: post
title:  "Build a Voice Bot using FreeSWITCH and Whisper"
date:   2023-04-01 23:11:00 +0500
categories: freeswitch openai-whisper speechbrain 
---
    
### Introduction

`mod_whisper` is a highly flexible and efficient solution designed specifically for seamlessly integrating FreeSWITCH with your ASR (Automatic Speech Recognition) and TTS (Text-to-Speech) Service using websockets. By leveraging `mod_whisper`, the process of integrating your existing ASR and TTS service over `websockets` becomes remarkably straightforward and time-efficient, setting it apart from the traditional MRCP (Media Resource Control Protocol) based ASR and TTS modules.

With `mod_whisper`, you can effortlessly establish a smooth connection between FreeSWITCH and your ASR and TTS service, empowering your applications with powerful voice recognition and generation capabilities. By following the comprehensive guidelines provided in the code repository, you can easily implement ASR and TTS services in Python, ensuring a reliable and reference-driven integration process.

By adopting `mod_whisper`, you can streamline the integration of your ASR and TTS services, enabling enhanced voice-based communication within your applications.


### How to setup 

[View README.md](https://github.com/cyrenity/mod_whisper) to see the installation steps 

Create the following sample dialplan to test both ASR and TTS functionality


```xml
    <extension name="asr_demo">
        <condition field="destination_number" expression="1888">
		<action application="answer"/>
        <action application="set" data="tts_engine=whisper"/>
        <action application="set" data="tts_voice=mustafa"/>
		<action application="sleep" data="500" />
		<action application="play_and_detect_speech" data="say:Do you want takeaway or home delivery? detect:whisper {channel-uuid=${uuid},start-input-timers=false}takeaway or delivery"/>
		<action application="log" data="CRIT ${detect_speech_result}!"/>-->
        </condition>
    </extension>

```

You can create more advanced dialplans with ESL and scripts in various languages. See examples in scripts folder.


## Important Links

- [FreeSWITCH](https://freeswitch.org)
- [Whisper](https://openai.com/research/whisper)
- [Mod_Whisper](https://github.com/cyrenity/mod_whisper)