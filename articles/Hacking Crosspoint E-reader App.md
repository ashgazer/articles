


curl --url 'http://crosspoint.local/api/status' \
  -H 'Accept: */*' \
  -H 'Accept-Language: en-GB,en-US;q=0.9,en;q=0.8' \
  -H 'Connection: keep-alive' \
  -H 'Referer: http://crosspoint.local/files' \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/152.0.0.0 Safari/537.36' \
  --insecure





curl --url 'http://crosspoint.local/api/status' \
  -H 'Accept: */*' \
  -H 'Accept-Language: en-GB,en-US;q=0.9,en;q=0.8' \
  -H 'Connection: keep-alive' \
  -H 'Referer: http://crosspoint.local/files?path=%2FArticles' \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/152.0.0.0 Safari/537.36' \
  --insecure




```
pip install websocket-client requests
python3 upload_to_crosspoint.py /path/to/folder
python3 upload_to_crosspoint.py /path/to/folder --path /Articles --pattern "*.epub"
Defaults: --host crosspoint.local, --path /Articles, --pattern * (all files), --http-port 80 (WS port auto-computed as http-port + 1, matching the device's own convention).

```
```

python3 upload_to_crosspoint.py /Users/ashik.pirmohamed/Documents/send_to_ereader --path /Articles --pattern "*.epub"
```