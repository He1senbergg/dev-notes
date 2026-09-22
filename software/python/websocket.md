# 新增websocket

下载`pip.pyz`、`websockets-17.0.1-py3-none-any.whl`

```
mkdir -p ~/tmp/websockets_offline
cd ~/tmp/websockets_offline
python3 -m pip download \
  websockets==17.0.1 \
  --no-deps \
  --only-binary=:all: \
  --platform any \
  --python-version 3.11 \
  -d .

curl -L https://bootstrap.pypa.io/pip/pip.pyz -o pip.pyz
```

把下好的`pip.pyz`、`websockets-17.0.1-py3-none-any.whl`，传到机器上。

安装一下

```
mkdir -p pydeps
python pip.pyz install \
  --no-index \
  --no-deps \
  --target ./pydeps \
  websockets-17.0.1-py3-none-any.whl
```

验证一下`websockets`安装好了没

```
PYTHONPATH=$PWD/pydeps python -c \
"import asyncio, json, websockets; print('all OK'); print(websockets.__version__)"
```

![image](../../png/software/18.png)

跑两个脚本

```
cat <<'EOF' > TestWs.py
import asyncio
import json
import websockets

SOCKET = "/opt/sharevolume/uds/iminferscheduler.sock"

async def main():
    async with websockets.unix_connect(
        SOCKET,
        uri="ws://localhost/v1/chat/completions",
        subprotocols=["iminfer-ws"],
    ) as ws:
        await ws.send(json.dumps({
            "model": "Qwen3_8B_w8a8s",
            "messages": [
                {"role": "user", "content": "hello"}
            ],
            "stream": False
        }))

        print(await ws.recv())
asyncio.run(main())
EOF
```

```
cat <<'EOF' > StreamTestWs.py
import asyncio
import json
import websockets

SOCKET = "/opt/sharevolume/uds/iminferscheduler.sock"


async def main():
    async with websockets.unix_connect(
        SOCKET,
        uri="ws://localhost/v1/chat/completions",
        subprotocols=["iminfer-ws"],
    ) as ws:

        request = {
            "model": "Qwen3_8B_w8a8s",
            "messages": [
                {"role": "user", "content": "hello"}
            ],
            "stream": True
        }

        await ws.send(json.dumps(request))

        while True:
            try:
                message = await ws.recv()

                print(message)

                if message == "[DONE]":
                    break

            except websockets.ConnectionClosed:
                print("WebSocket connection closed")
                break


asyncio.run(main())
EOF
```