---
name: twmd-clients-multilang
description: >-
  Minimal TW Market Data (TWMD) API clients in languages beyond Python/JS — Go,
  C#/.NET, Java, Julia, Ruby, PHP, and a raw curl reference. Use when the user
  works in one of these languages and wants a starter client for TWMD (GET
  /v2/datasets/{id} with the X-API-Key header), or asks "TWMD in Go/C#/Java/Julia/
  Ruby/PHP." Each snippet fetches a dataset and returns the rows. Key in the
  header, never the URL.
---

# TWMD clients — multi-language

Starter clients for `GET https://api.twmarketdata.com/v2/datasets/{id}` with header `X-API-Key`. **Not investment advice.** Key in env, never in the URL.

## curl (reference)
```bash
curl "https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10" \
  -H "X-API-Key: $TWMD_API_KEY"
```

## Go
```go
package main
import ("encoding/json";"fmt";"net/http";"os";"time")
func main() {
    req, _ := http.NewRequest("GET",
        "https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10", nil)
    req.Header.Set("X-API-Key", os.Getenv("TWMD_API_KEY"))
    resp, err := (&http.Client{Timeout: 30 * time.Second}).Do(req)
    if err != nil { panic(err) }
    defer resp.Body.Close()
    var body struct{ Data []map[string]any `json:"data"` }
    json.NewDecoder(resp.Body).Decode(&body)
    fmt.Println(len(body.Data), "rows")
}
```

## C# / .NET
```csharp
using System.Net.Http; using System.Text.Json;
var http = new HttpClient();
http.DefaultRequestHeaders.Add("X-API-Key", Environment.GetEnvironmentVariable("TWMD_API_KEY"));
var url = "https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10";
var json = await http.GetStringAsync(url);
using var doc = JsonDocument.Parse(json);
var rows = doc.RootElement.GetProperty("data");
Console.WriteLine($"{rows.GetArrayLength()} rows");
```

## Java (11+)
```java
import java.net.http.*; import java.net.URI;
var req = HttpRequest.newBuilder(URI.create(
    "https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10"))
    .header("X-API-Key", System.getenv("TWMD_API_KEY")).build();
var resp = HttpClient.newHttpClient().send(req, HttpResponse.BodyHandlers.ofString());
System.out.println(resp.body());   // parse JSON (Jackson/Gson) → body.data
```

## Julia
```julia
using HTTP, JSON3
r = HTTP.get("https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10",
             ["X-API-Key" => ENV["TWMD_API_KEY"]])
data = JSON3.read(String(r.body)).data
println(length(data), " rows")
```

## Ruby
```ruby
require "net/http"; require "json"
uri = URI("https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10")
req = Net::HTTP::Get.new(uri); req["X-API-Key"] = ENV["TWMD_API_KEY"]
res = Net::HTTP.start(uri.host, uri.port, use_ssl: true) { |h| h.request(req) }
puts JSON.parse(res.body)["data"].length
```

## PHP
```php
<?php
$ctx = stream_context_create(["http" => ["header" => "X-API-Key: ".getenv("TWMD_API_KEY")]]);
$json = file_get_contents(
  "https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10", false, $ctx);
$data = json_decode($json, true)["data"];
echo count($data), " rows\n";
```

## Shared notes
- Envelope: `{ dataset, source_role, freshness, lineage.trace_id, data_gaps, data:[...] }`.
- Handle 401 (bad key), 402 (not entitled/quota), 429 (rate) — back off on 429.
- Confirm dataset ids at `https://twmarketdata.com/llms.txt`, fields at `<id>.md` / `/openapi.json`.

## Honesty
Not investment advice. Daily + fundamentals. Key in the header, never the URL/logs.
