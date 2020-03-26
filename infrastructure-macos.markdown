# Builders for macOS
{:mac-checkers}

There is a dedicated process to run validation of pullrequests for macos.The routines therein are implemented within a [runscript](https://github.com/alisw/ali-bot/blob/master/ci/run-continuous-builder.sh) inside [ali-bot](https://github.com/alisw/ali-bot). This script currently needs to be started manually using a SREEN session to ensure it stays open. A launchd version is wip. 

```bash
SCREEN -dmS <session_name> /build/ali-bot/ci/run-continuous-builder.sh <config_file_argument>
```
There are four dedicated worker nodes:
* `alibuildmac01` `alibuildmac02` process pull requests for o2 and are launched by providing argument [`o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/o2-0.sh) resp. [`o2-1`](https://github.com/alisw/ali-bot/blob/master/ci/conf/o2-1.sh)
* `alibuildmac03` `alibuildmac04` process pull requests for alidist and are launched by providing argument [`alidist_o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/alidist_o2-0.sh) and [`alidist_o2-1`](https://github.com/alisw/ali-bot/blob/master/ci/conf/alidist_o2-1.sh)
* the actual builds are taking place in `/build/<config_file_argument>` 
