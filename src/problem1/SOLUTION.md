Simple command to accomplish the task:

```bash
jq -r 'select(.symbol == "TSLA" and .side == "sell") | .order_id' ./transaction-log.txt | xargs -P 10 -I{} curl -s --max-time 30 "https://example.com/api/{}" > ./output.txt 
```
What this command does:
- Filter the transaction file with jq to get the right order_id, assuming the jq tool already installed in the host 
- Run upto 10 curl processes in parallel to speed up the command 

---
Considering the transaction file getting larger with GB+ of data, using awk instead of jg to reduce Json processing overhead: 

```bash
awk -F'"' '/"symbol": "TSLA"/ && /"side": "sell"/ {print $4}' ./transaction-log.txt |  xargs -P 10 -I{} curl -s --max-time 30 "https://example.com/api/{}" > ./output.txt 
```

Assuming the transaction file format unchanged accross entries