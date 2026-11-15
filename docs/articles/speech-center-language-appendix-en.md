# Speech Center Appendix: Licensing and Languages

> OpenMaas Engineering Series · Speech Center Appendix
>
> By Meng (Solomon) XU
>
> Version 2.0 | 2026-11-14

[Back to the article](./speech-center-article-en.md)

**Jump to a region:** [Common](#common) · [Asia](#asian) · [South Asia](#south-asian) · [Europe](#european) · [Middle East](#middle-east)

<a id="glossary"></a>
## Glossary

| Term | Meaning |
|---|---|
| Speech recognition (ASR) | Turning audio into text |
| Speech synthesis (TTS) | Turning text into audio |
| Voice | The identity of a synthesized voice |
| Worker | The process that executes recognition or synthesis |
| Lease | A temporary claim on worker capacity |
| Object storage | Storage for uploaded and generated audio |
| Retention window | How long a result remains available |
| Character error rate (CER) | Character-level difference from a reference transcript |
| Word error rate (WER) | Word-level difference from a reference transcript |
| Commercial use | Code, model weights, and voices all allow commercial use |

<a id="license-matrix"></a>
## License matrix

| Asset | Code license | Weights / voices | Commercial | Decision |
|---|---|---|---|---|
| Kokoro-82M-v1.1-zh | Apache-2.0 | Apache-2.0 | Yes | Default Chinese engine |
| Piper 1.2.0 / 1.8.0 | 1.2.0 is MIT; 1.8.0 is GPL-3.0-or-later | Per voice | Depends on voice | Version and distribution need legal review |
| piper-phonemize / espeak-ng | MIT wrapper + embedded GPLv3 | — | Legal review | Not a pure MIT distribution |
| Qwen3-TTS | Apache-2.0 initial review | Apache-2.0 initial review | Pending final review | ROCm and Jetson No-Go; CUDA candidate |
| CosyVoice2-0.5B | Apache-2.0 | Apache-2.0 | Yes | Deployment validated; not in the first release |
| XTTS-v2 | Coqui Public Model License 1.0.0 | Non-commercial | No | Not a default commercial engine |

Recognition uses the same split: SenseVoice/FunASR and faster-whisper enter the service-binding checklist with separate code and weight licenses. Engines marked non-commercial stay outside the default commercial set. The release list is the source of truth for SPDX identifiers.

<a id="language-list"></a>
## Language list

Notes:

- Language names follow Unicode CLDR.
- Speaker counts are order-of-magnitude estimates, mixing first and second language speakers.
- Ranges are used instead of exact counts because the sources differ in scope and year.
- “Main regions” is representative, not a political boundary.
- TTS and ASR status is a snapshot for this article.

**Status:**

- Ready: passed the quality gate.
- Degraded: the pipeline works; business quality is still under evaluation.
- Trial: available to a limited audience; sample coverage is thin.
- Non-commercial: technically available; commercial use needs a license.
- Hidden: registered in the back end but not shown in the catalog.

> Speaker counts are approximate, mix L1 and L2 speakers, and are not a population ranking.

<a id="common"></a>
### Common

| Code | English name | Native name | Main regions | Speakers | TTS | ASR | Commercial |
|---|---|---|---|---|---|---|---|
| zh-CN | Chinese (Simplified) | 简体中文 | China, Singapore, overseas Chinese | >1B | Ready | Ready | Yes |
| en | English | English | US, UK, Canada, Australia, India, etc. | >1B | Ready | Ready | Yes |

<a id="asian"></a>
### Asia

| Code | English name | Native name | Main regions | Speakers | TTS | ASR | Commercial |
|---|---|---|---|---|---|---|---|
| ja | Japanese | 日本語 | Japan | 100M–1B | Ready | Degraded | Yes |
| ko | Korean | 한국어 | South Korea, North Korea, Yanbian | 10M–100M | Ready | Degraded | Yes |
| yue | Cantonese | 粤语 | Guangdong, Hong Kong, Macau, overseas Chinese | 10M–100M | — | Degraded | — |
| kk | Kazakh | Қазақша | Kazakhstan, Xinjiang | 10M–100M | Degraded | — | Yes |
| vi | Vietnamese | Tiếng Việt | Vietnam, diaspora | 10M–100M | Degraded | Degraded | Yes |
| id | Indonesian | Bahasa Indonesia | Indonesia | 100M–1B | — | — | — |
| ms | Malay | Bahasa Melayu | Malaysia, Brunei, Singapore | 10M–100M | — | — | — |
| th | Thai | ไทย | Thailand | 10M–100M | Non-commercial | Degraded | No |

<a id="south-asian"></a>
### South Asia

| Code | English name | Native name | Main regions | Speakers | TTS | ASR | Commercial |
|---|---|---|---|---|---|---|---|
| hi | Hindi | हिन्दी | India, Fiji, diaspora | 100M–1B | Non-commercial | Degraded | No |
| bn | Bengali | বাংলা | Bangladesh, West Bengal | 100M–1B | Degraded | Trial (hidden) | Yes |
| ur | Urdu | اردو | Pakistan, India | 100M–1B | Degraded | Trial | Yes |
| ne | Nepali | नेपाली | Nepal, India | 10M–100M | Degraded | Degraded | Yes |
| mr | Marathi | मराठी | Maharashtra, India | 10M–100M | Degraded | Degraded | Yes |
| te | Telugu | తెలుగు | Andhra Pradesh, Telangana | 10M–100M | Degraded | Trial (hidden) | Yes |

<a id="european"></a>
### Europe

| Code | English name | Native name | Main regions | Speakers | TTS | ASR | Commercial |
|---|---|---|---|---|---|---|---|
| fr | French | Français | France, Canada, Africa | 100M–1B | Ready | Degraded | Yes |
| de | German | Deutsch | Germany, Austria, Switzerland | 100M–1B | Ready | Degraded | Yes |
| es | Spanish | Español | Spain, Latin America, US | 100M–1B | Ready | Degraded | Yes |
| ru | Russian | Русский | Russia, CIS | 100M–1B | Ready | Degraded | Yes |
| pt-BR | Portuguese (Brazil) | Português (Brasil) | Brazil, Portugal, diaspora | 100M–1B | Ready | Degraded | Yes |
| it | Italian | Italiano | Italy, Switzerland | 10M–100M | Degraded | Degraded | Yes |
| nl | Dutch | Nederlands | Netherlands, Belgium | 10M–100M | Degraded | Degraded | Yes |
| pl | Polish | Polski | Poland | 10M–100M | Degraded | Degraded | Yes |
| cs | Czech | Čeština | Czechia | 10M–100M | Degraded | Degraded | Yes |
| sv | Swedish | Svenska | Sweden, Finland | 10M–100M | Degraded | Degraded | Yes |
| da | Danish | Dansk | Denmark | <10M | Degraded | Degraded | Yes |
| nb | Norwegian Bokmål | Norsk bokmål | Norway | <10M | Degraded | Degraded | Yes |
| uk | Ukrainian | Українська | Ukraine, diaspora | 10M–100M | Degraded | Degraded | Yes |
| bg | Bulgarian | Български | Bulgaria | <10M | Degraded | Degraded | Yes |
| el | Greek | Ελληνικά | Greece, Cyprus | 10M–100M | Degraded | Degraded | Yes |
| ca | Catalan | Català | Catalonia, Andorra | 10M–100M | Degraded | Degraded | Yes |
| fi | Finnish | Suomi | Finland | <10M | Degraded | Degraded | Yes |
| hu | Hungarian | Magyar | Hungary, neighbors | 10M–100M | Degraded | Degraded | Yes |
| ro | Romanian | Română | Romania, Moldova | 10M–100M | Degraded | Degraded | Yes |
| sk | Slovak | Slovenčina | Slovakia | <10M | Degraded | Degraded | Yes |
| sl | Slovenian | Slovenščina | Slovenia | <10M | Degraded | Trial | Yes |
| sq | Albanian | Shqip | Albania, Kosovo, North Macedonia | <10M | Degraded | Degraded | Yes |
| sr | Serbian | Српски | Serbia, Bosnia | <10M | Non-commercial | Trial | No |
| lb | Luxembourgish | Lëtzebuergesch | Luxembourg | <10M | Non-commercial | Degraded | No |
| cy | Welsh | Cymraeg | Wales | <10M | Degraded | — | Yes |
| et | Estonian | Eesti | Estonia | <10M | Degraded | — | Yes |
| eu | Basque | Euskara | Basque Country, southern France | <10M | Degraded | — | Yes |
| lv | Latvian | Latviešu | Latvia | <10M | Degraded | — | Yes |

<a id="middle-east"></a>
### Middle East

| Code | English name | Native name | Main regions | Speakers | TTS | ASR | Commercial |
|---|---|---|---|---|---|---|---|
| ar | Arabic | العربية | Arab countries, diaspora | 100M–1B | Ready | Degraded | Yes |
| fa | Persian | فارسی | Iran, Afghanistan, Tajikistan | 100M–1B | Degraded | Degraded | Yes |
| he | Hebrew | עברית | Israel | <10M | Non-commercial | Degraded | No |
| tr | Turkish | Türkçe | Turkey, Cyprus | 10M–100M | Degraded | Degraded | Yes |
| ku | Kurdish | Kurdî | Turkey, Iraq, Syria, Iran | 10M–100M | Non-commercial | — | No |

### Not in the shared catalog

`lt`, `hy`, `is`, `ka`, `ml`, and `sw` are not in the shared language catalog. Reasons include missing phonemizer support, unverified source or license, or no available voice.

## Notes

- Speaker data comes from Unicode CLDR, Ethnologue, and public statistical sources; years and methods differ.
- Counts combine L1 and L2 speakers and are suitable for coverage planning, not exact ranking.
- Language status changes with new samples and licenses; the current service catalog is the source of truth.
- Non-commercial voices and language packs need separate rights before use in a commercial project.
