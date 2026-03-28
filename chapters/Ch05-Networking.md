<!-- Windows 11 Troubleshooting Bible — Network Stack Troubleshooting -->

## 13. Network Stack Troubleshooting

### The Complete Network Reset Sequence

Run in this exact order from admin cmd.exe:

```cmd
:: Step 1: Reset Winsock (network socket layer)
netsh winsock reset catalog

:: Step 2: Reset TCP/IP stack
netsh int ip reset

:: Step 3: Reset IPv6 stack
netsh int ipv6 reset

:: Step 4: Flush DNS cache
ipconfig /flushdns

:: Step 5: Re-register DNS
ipconfig /registerdns

:: Step 6: Release current IP
ipconfig /release

:: Step 7: Renew IP
ipconfig /renew

:: Step 8: REBOOT (required for winsock/TCP/IP reset to take effect)
shutdown /r /t 0
```

### What Each Command Does

| Command | What It Resets | When To Use |
|---------|---------------|-------------|
| `netsh winsock reset` | Winsock catalog (socket API) | After malware, bad network drivers, VPN uninstall |
| `netsh int ip reset` | TCP/IP stack registry entries | "Limited connectivity", DHCP failures, IP conflicts |
| `ipconfig /flushdns` | Local DNS cache | Stale DNS entries, can't reach websites that recently changed IPs |
| `ipconfig /registerdns` | Forces DNS re-registration | Computer not visible on network |
| `ipconfig /release` + `/renew` | DHCP lease | Get a fresh IP address |

### DNS Troubleshooting

```cmd
:: Check current DNS servers
ipconfig /all | findstr "DNS Servers"

:: Test DNS resolution
nslookup google.com
nslookup google.com 8.8.8.8  :: Test with Google's DNS directly

:: If first fails but second works → your DNS server is the problem
:: Fix: Change DNS to 8.8.8.8 / 8.8.4.4 (Google) or 1.1.1.1 / 1.0.0.1 (Cloudflare)
```

### Proxy Issues

```cmd
:: Check system proxy settings
netsh winhttp show proxy

:: Reset proxy
netsh winhttp reset proxy

:: Check environment proxy variables
echo %HTTP_PROXY%
echo %HTTPS_PROXY%
echo %NO_PROXY%
```

### Firewall Troubleshooting

```cmd
:: Check Windows Firewall status
netsh advfirewall show allprofiles state

:: Temporarily disable (TESTING ONLY)
netsh advfirewall set allprofiles state off

:: Re-enable
netsh advfirewall set allprofiles state on

:: Check if a specific port is allowed
netsh advfirewall firewall show rule name=all dir=in | findstr "8080"

:: Add a firewall rule
netsh advfirewall firewall add rule name="Docker" dir=in action=allow protocol=tcp localport=2375
```

---

