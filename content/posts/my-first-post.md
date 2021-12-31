---
title: "My First Post"
date: 2021-12-31T11:29:57+03:00
draft: false
---

deneme

{{< highlight python >}}
async def do_http_req():
    async with aiohttp.ClientSession() as session:
        async with session.get('http://python.org') as response:
            print("Status:", response.status)
            print("Content-type:", response.headers['content-type'])

            html = await response.text()
            print("Body:", html[:15], "...")
{{< / highlight >}}
