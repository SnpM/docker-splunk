
```
sudo docker run -d \
  --name splunkreb2 \
  -e SPLUNK_START_ARGS="--accept-license" \
  -e SPLUNK_PASSWORD="changeme123" \
  -p 8000:8000\
  -p 8089:8089 \
  -v ~/splunk_image/splunk/etc:/opt/splunk/etc \
  -v ~/splunk_image/splunk/var:/opt/splunk/var \
  splunk-rebase:latest
```