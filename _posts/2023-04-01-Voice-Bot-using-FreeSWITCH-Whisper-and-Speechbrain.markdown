---
layout: post
title:  "Build a Voice Bot using FreeSWITCH and Whisper"
date:   2023-04-01 23:11:00 +0500
categories: freeswitch voip telephony ai speech asr tts openai/whisper speechbrain
---
    
### Introduction

`mod_whisper` is a custom FreeSWITCH to integrate FreeSWITCH with your ASR & TTS Service over websockets.

Unlike `mrcp` based ASR && TSS modules, `mod_whisper` makes it simple and quick to integrate your existing ASR and TTS service over websockets 

Follow the guidelines in code repository for refernce implementation of ASR & TTS service in python
 
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

