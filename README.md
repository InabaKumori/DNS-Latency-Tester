# DNS Latency Tester

This script tests the latency of public DNS servers in your country and provides the fastest IPv4 and IPv6 addresses. It retrieves your public IP address, determines your country code, fetches a list of public DNS servers for your country, and performs a ping test on each server to measure the latency. The results are sorted by latency in ascending order and saved to a file named `dns.txt`.

## Prerequisites

- `curl` command-line tool
- `ping` command-line tool
- `awk` command-line tool
- `sort` command-line tool

## Usage

### Method 1: Direct Terminal Input

1. Open your terminal.
2. Copy and paste the following commands into your terminal and press Enter:

```bash
cat << 'EOF' > /tmp/dns.sh
#!/bin/bash

my_ip=$(curl -s https://api.ipify.org)
if [ -n "$my_ip" ]; then
    country_code=$(curl -s "https://ipinfo.io/${my_ip}/country" | tr -d '"')
    if [ -n "$country_code" ]; then
        country_code=$(echo "$country_code" | tr '[:upper:]' '[:lower:]')
        dns_list_url="https://public-dns.info/nameserver/${country_code}.txt"
        dns_ips=$(curl -s "$dns_list_url")
        if [ -n "$dns_ips" ]; then
            input_file=$(mktemp)
            echo "$dns_ips" > "$input_file"
            output_file="dns.txt"
            tmp_file=$(mktemp)

            function ping_ip {
                local ip="$1"
                local result=$(ping -c 3 -i 0.2 -W 1 "$ip" | grep 'min/avg/max' | awk -F'/' '{print $5}')
                if [ -n "$result" ]; then
                    echo "$ip $result" >> "$tmp_file"
                fi
            }

            # 并发PING所有IP
            while IFS= read -r ip || [[ -n "$ip" ]]; do
                ping_ip "$ip" &
                if (( $(jobs -p | wc -l) >= 100 )); then
                    wait
                fi
            done < "$input_file"
            wait

            # 延迟结果按照延迟升序排序并写入dns.txt
            if [ -s "$tmp_file" ]; then
                sort -n -k2 "$tmp_file" | awk '{print $1, $2}' > "$output_file"
            else
                echo "所有 IP 地址均超时，未能获取有效结果。" > "$output_file"
            fi

            rm "$tmp_file"
            rm "$input_file"

            # 从dns.txt获取延迟最低的IPv4和IPv6
            best_ip_v4=$(awk '/\./ {print $1; exit}' "$output_file")
            best_ip_v6=$(awk -F'[ \t]+|/' '/:/{print $1; exit}' "$output_file")

            # 显示延迟最低的IPv4和IPv6（仅在SSH会话中）
            if [ -n "$best_ip_v4" ]; then
                echo "选择延迟最低的 IPv4 地址为: $best_ip_v4"
            else
                echo "无法获取延迟最低的 IPv4 地址。"
            fi

            if [ -n "$best_ip_v6" ]; then
                echo "选择延迟最低的 IPv6 地址为: $best_ip_v6"
            else
                echo "无法获取延迟最低的 IPv6 地址。"
            fi

            echo "Ping 测试完成，并将结果按延迟排序保存到 $output_file 文件中。"
        else
            echo "无法获取国家 ${country_code} 的公共DNS服务器列表。"
        fi
    else
        echo "无法获取本机 IP 地址 ${my_ip} 所属国家代码。"
    fi
else
    echo "无法获取本机公网 IP 地址。"
fi
EOF

chmod +x /tmp/dns.sh && /tmp/dns.sh
```

3. The script will execute and display the fastest IPv4 and IPv6 addresses. The complete results will be saved in a file named `dns.txt` in the current directory.

### Method 2: Running the Script

1. Create a new file named `dns_latency_tester.sh` and open it in a text editor.
2. Copy and paste the following code into the file:

```bash
#!/bin/bash

my_ip=$(curl -s https://api.ipify.org)
if [ -n "$my_ip" ]; then
    country_code=$(curl -s "https://ipinfo.io/${my_ip}/country" | tr -d '"')
    if [ -n "$country_code" ]; then
        country_code=$(echo "$country_code" | tr '[:upper:]' '[:lower:]')
        dns_list_url="https://public-dns.info/nameserver/${country_code}.txt"
        dns_ips=$(curl -s "$dns_list_url")
        if [ -n "$dns_ips" ]; then
            input_file=$(mktemp)
            echo "$dns_ips" > "$input_file"
            output_file="dns.txt"
            tmp_file=$(mktemp)

            function ping_ip {
                local ip="$1"
                local result=$(ping -c 3 -i 0.2 -W 1 "$ip" | grep 'min/avg/max' | awk -F'/' '{print $5}')
                if [ -n "$result" ]; then
                    echo "$ip $result" >> "$tmp_file"
                fi
            }

            # 并发PING所有IP
            while IFS= read -r ip || [[ -n "$ip" ]]; do
                ping_ip "$ip" &
                if (( $(jobs -p | wc -l) >= 100 )); then
                    wait
                fi
            done < "$input_file"
            wait

            # 延迟结果按照延迟升序排序并写入dns.txt
            if [ -s "$tmp_file" ]; then
                sort -n -k2 "$tmp_file" | awk '{print $1, $2}' > "$output_file"
            else
                echo "所有 IP 地址均超时，未能获取有效结果。" > "$output_file"
            fi

            rm "$tmp_file"
            rm "$input_file"

            # 从dns.txt获取延迟最低的IPv4和IPv6
            best_ip_v4=$(awk '/\./ {print $1; exit}' "$output_file")
            best_ip_v6=$(awk -F'[ \t]+|/' '/:/{print $1; exit}' "$output_file")

            # 显示延迟最低的IPv4和IPv6（仅在SSH会话中）
            if [ -n "$best_ip_v4" ]; then
                echo "选择延迟最低的 IPv4 地址为: $best_ip_v4"
            else
                echo "无法获取延迟最低的 IPv4 地址。"
            fi

            if [ -n "$best_ip_v6" ]; then
                echo "选择延迟最低的 IPv6 地址为: $best_ip_v6"
            else
                echo "无法获取延迟最低的 IPv6 地址。"
            fi

            echo "Ping 测试完成，并将结果按延迟排序保存到 $output_file 文件中。"
        else
            echo "无法获取国家 ${country_code} 的公共DNS服务器列表。"
        fi
    else
        echo "无法获取本机 IP 地址 ${my_ip} 所属国家代码。"
    fi
else
    echo "无法获取本机公网 IP 地址。"
fi
```

3. Save the file and close the text editor.
4. Open your terminal and navigate to the directory where you saved the script.
5. Make the script executable by running the following command:

```bash
chmod +x dns_latency_tester.sh
```

6. Run the script using the following command:

```bash
./dns_latency_tester.sh
```

7. The script will execute and display the fastest IPv4 and IPv6 addresses. The complete results will be saved in a file named `dns.txt` in the current directory.

## Notes

- This script has been successfully tested on macOS and Linux.
- Make sure you have the required command-line tools (`curl`, `ping`, `awk`, `sort`) installed on your system.
- The script retrieves your public IP address and determines your country code to fetch the appropriate list of public DNS servers.
- The ping test is performed concurrently, with a maximum of 100 concurrent pings at a time.
- The results are sorted by latency in ascending order and saved to the `dns.txt` file.
- If all IP addresses time out during the ping test, an appropriate message will be displayed and saved to the `dns.txt` file.

## License

This project is licensed under the GNU General Public License v3.0. See the [LICENSE](LICENSE) file for details.

## Author

- [inabakumori](https://github.com/inabakumori)

Feel free to contribute, report issues, or provide feedback!
