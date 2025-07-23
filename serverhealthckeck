#!/usr/bin/env python3

from scapy.all import ICMP, IP, sr1
import pandas as pd
import time
from datetime import datetime, timezone
from melipayamak import Api
from concurrent.futures import ThreadPoolExecutor
from colorama import init, Fore
import logging
from elasticsearch import Elasticsearch 
import urllib3 
from collections import deque
import os
import json
import socket
from jdatetime import date as jalaliDate

# Disable SSL Alarams
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# For Color in Line
init(autoreset=True)

config_file = 'server_config.json'

def get_server_ip():
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
            s.connect(("8.8.8.8", 80))
            server_ip = s.getsockname()[0]
        return server_ip
    except Exception as e:
        logging.error(f"Failed to retrieve server IP: {e}")
        return "127.0.0.1"

def get_or_create_alias():
    if os.path.exists(config_file):
        with open(config_file, 'r') as file:
            config = json.load(file)
            return config.get('alias_name'), config.get('server_ip')
    else:
        alias_name = input("Please enter an Alias Name for this server: ").strip()
        server_ip = get_server_ip()
        config = {
            'alias_name': alias_name,
            'server_ip': server_ip
        }
        with open(config_file, 'w') as file:
            json.dump(config, file, indent=4)
        return alias_name, server_ip

alias_name, server_ip = get_or_create_alias()

# Setup Logging
logging.basicConfig(
    filename="ping_logs.log",
    level=logging.ERROR,
    format="%(asctime)s - %(levelname)s - %(message)s"
)

# Connect to Elastic Server Kibana
es = Elasticsearch('https://kibana-server:9200',
                   basic_auth=('USERNAME', 'PASSWORD'),
                   verify_certs=False,
                   ssl_show_warn=False)




# Information For SMS Panel
username = 'USERNAME'
password = 'PASSWORD'
api = Api(username, password)
sms = api.sms()
_from = 'number of sms pannel'
rec = ['Num3', 'Num2','Num3']
latency_check_queue = deque()

status_file = "ip_status.json"
ip_status_dict = {}

def load_status_from_file():
    global ip_status_dict
    if os.path.exists(status_file):
        with open(status_file, "r") as f:
            ip_status_dict = json.load(f)
    else:
        ip_status_dict = {}

def save_status_to_file():
    with open(status_file, "w") as f:
        json.dump(ip_status_dict, f, indent=4)

def reset_status():
    for ip, data in ip_status_dict.items():
        data["dead"] = 0
        data["semi_dead"] = 0
        data["avg_latency_count"] = 0
        data["alive"] = 0

def read_addresses_from_excel(file_path):
    df = pd.read_excel(file_path)
    return df[['IP', 'Name']]

def ping_once(address):
    packet = IP(dst=address)/ICMP()
    start_time = time.time()
    response = sr1(packet, timeout=1, verbose=False)
    end_time = time.time()

    if response:
        latency = (end_time - start_time) * 1000
        return {"address": address, "status": "alive", "time": latency, "latency": latency}
    else:
        return {"address": address, "status": "timeout", "time": None, "latency": None}

def ping_address(address):
    times = []
    latencies = []
    for _ in range(4):
        result = ping_once(address)
        if result["status"] == "alive":
            times.append(result["time"])
            latencies.append(result["latency"])
        else:
            times.append(None)
            latencies.append(None)
        time.sleep(0.1)

    status = "dead" if len([time for time in times if time is not None]) == 0 else "alive"
    valid_latencies = [latency for latency in latencies if latency is not None]
    avg_latency = sum(valid_latencies) / len(valid_latencies) if valid_latencies else None
    avg_latency = round(avg_latency, 2) if avg_latency is not None else None
    packet_loss = 100 * (4 - len(valid_latencies)) / 4

    if packet_loss >= 100:
        status = "dead"
    elif 50 <= packet_loss <= 75:
        status = "semi_dead"
    elif packet_loss < 50:
        status = "alive"

    return {
        "address": address,
        "times": times,
        "status": status,
        "latencies": latencies,
        "avg_latency": avg_latency,
        "packet_loss": packet_loss,
        "sms_sent": False
    }

def send_sms(ip, name, message_template):
    now_time = datetime.now().strftime("%H:%M:%S")
    today_date_persian = str(jalaliDate.today())
    try:
        message = message_template.format(ip=ip, name=name, now_time=now_time, today_date_persian=today_date_persian)
        logging.info(f"Sending SMS for {ip} ({name}) at {now_time} on {today_date_persian}")
        all_sms_sent = True
        for contact in rec:
            response = sms.send(contact, _from, message)
            print(f"Response for {contact}: {response}")
            logging.info(f"SMS response for {contact}: {response}")
            if not response:
                all_sms_sent = False
        return all_sms_sent
    except Exception as e:
        logging.error(f"Error sending SMS for {ip} ({name}): {e}")
        return False

def update_ip_status(result):
    ip = result["address"]
    name = result["name"]
    status = result["status"]
    avg_latency = result["avg_latency"]

    if ip not in ip_status_dict:
        ip_status_dict[ip] = {
            "dead": 0,
            "semi_dead": 0,
            "alive": 0,
            "avg_latency_count": 0,
            "sent_sms_count": 0,
            "last_sms_times": [],
        }

    data = ip_status_dict[ip]

    if avg_latency is not None:
        logging.info(f"avg_latency for {ip}: {avg_latency}")
        if avg_latency > 250:
            logging.info(f"Increasing avg_latency_count for {ip} (Before: {data['avg_latency_count']})")
            data["avg_latency_count"] += 1
            if data["avg_latency_count"] == 15:
                message_template = "🟧 آی‌پی {ip} ({name}) دارای نرخ تاخیر پکت بالا است.\n\nزمان: {now_time} تاریخ: {today_date_persian}"
                sms_sent = send_sms(ip, name, message_template)
                if sms_sent:
                    result["sms_sent"] = True
                else:
                    result["sms_sent"] = False
        else:
            logging.info(f"avg_latency for {ip} is less than 250, not incrementing avg_latency_count")

    if status == "dead":
        data["dead"] += 1
        if data["dead"] >= 4 and data["sent_sms_count"] < 2:
            message_template = "🟥 آی‌پی {ip} ({name}) از دسترس خارج شده است و در صورت بازگشت اطلاع‌رسانی خواهد شد.\n\nزمان: {now_time} تاریخ: {today_date_persian}"
            sms_sent = send_sms(ip, name, message_template)
            if sms_sent:
                data["alive"] = 0
                data["sent_sms_count"] += 1
                result["sms_sent"] = True
            else:
                result["sms_sent"] = False

    elif status == "semi_dead":
        data["semi_dead"] += 1
        data["dead"] = 0
        if data["semi_dead"] == 20:
            message_template = "🟨 آی‌پی {ip} ({name}) دارای اختلال است و نیاز به بررسی دارد.\n\nزمان: {now_time} تاریخ: {today_date_persian}"
            sms_sent = send_sms(ip, name, message_template)
            if sms_sent:
                result["sms_sent"] = True
            else:
                result["sms_sent"] = False

    elif status == "alive":
        data["alive"] += 1
        data["dead"] = 0
        if data["alive"] >= 50 and data["sent_sms_count"] >= 1:
            message_template = "🟩 آی‌پی {ip} ({name}) به حالت نرمال بازگشته است.\n\nزمان: {now_time} تاریخ: {today_date_persian}"
            sms_sent = send_sms(ip, name, message_template)
            if sms_sent:
                data["sent_sms_count"] = 0
                data["dead"] = 0
                data["alive"] = 0
                result["sms_sent"] = True
            else:
                result["sms_sent"] = False

    save_status_to_file()

def calculate_statistics_for_address(result):
    times = result["times"]
    valid_times = [time for time in times if time is not None]

    if len(valid_times) == 0:
        return {
            "min": "N/A",
            "avg": "N/A",
            "max": "N/A",
            "jitter": "N/A",
            "packet_loss": 100
        }

    min_time = min(valid_times)
    max_time = max(valid_times)
    avg_time = sum(valid_times) / len(valid_times)
    jitter = (sum([(x - avg_time) ** 2 for x in valid_times]) / len(valid_times)) ** 0.5
    packet_loss = 100 if len(valid_times) == 0 else 100 * (4 - len(valid_times)) / 4

    return min_time, avg_time, max_time, jitter, packet_loss

def save_to_elasticsearch(result):
    
    if not es.indices.exists(index=index_name):
        es.indices.create(index=index_name)
        logging.info(f"Created new index: {index_name}")

    document = {
        "server_ip": server_ip,
        "alias_name": str(alias_name),
        "address": result['address'],
        "name": result['name'],
        "status": result['status'],
        "sms_sent": result['sms_sent'],
        "timestamp": datetime.now(timezone.utc).isoformat()
    }

    if result['status'] == 'dead':
        document.update({
            "packet_loss": 100,
        })
    elif result['status'] not in ['dead', 'timeout']:
        document.update({
            "times": result['times'],
            "min": result['min'],
            "avg": result['avg'],
            "max": result['max'],
            "jitter": float(result['jitter']) if isinstance(result['jitter'], (int, float)) else None,
            "packet_loss": result['packet_loss'],
            "avg_latency": result['avg_latency']
        })

    try:
        es.index(index=index_name, document=document)
        logging.info(f"Document indexed successfully for {result['address']} in {index_name}")
    except Exception as e:
        logging.error(f"Error indexing document for {result['address']}: {e}")

def ping_addresses(addresses, status_history):
    results = []
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = {executor.submit(ping_address, row['IP']): row['Name'] for _, row in addresses.iterrows()}
        for future in futures:
            try:
                result = future.result(timeout=5)
            except Exception as e:
                logging.error(f"Error while pinging {futures[future]}: {e}")
                continue

            result['name'] = futures[future]
            address = result['address']
            status = result['status']

            if address not in status_history:
                status_history[address] = {
                    "dead_count": 0,
                    "semi_dead_count": 0,
                    "sms_sent_count": 0,
                    "alive_count": 0,
                    "last_status": None
                }

            update_ip_status(result)

            min_time, avg_time, max_time, jitter, packet_loss = calculate_statistics_for_address(result)
            result["min"] = min_time
            result["avg"] = avg_time
            result["max"] = max_time
            result["jitter"] = jitter
            result["packet_loss"] = packet_loss

            save_to_elasticsearch(result)
            results.append(result)

    return results

if __name__ == "__main__":
    load_status_from_file()
    addresses = read_addresses_from_excel("IPs_For_Ping.xlsx")
    status_history = {}
    last_reset_time = time.time()
    current_date = datetime.now().strftime('%Y-%m-%d')
    index_name = f"ping_results-{current_date}"

    while True:
        new_date = datetime.now().strftime('%Y-%m-%d')
        if new_date != current_date:
            current_date = new_date
            index_name = f"ping_results-{current_date}"
            logging.info(f"Updated index name to {index_name}")

        print("Starting ping cycle...")
        ping_results = ping_addresses(addresses, status_history)

        for result in ping_results:
            sms_status_color = Fore.RED if result["sms_sent"] else Fore.BLUE
            sms_status = "YES" if result["sms_sent"] else "NO"

            status_color = Fore.GREEN if result["status"] == "alive" else Fore.YELLOW if result["status"] == "semi_dead" else Fore.RED
            print(f"Results for {result['name']} ({result['address']}):")
            print(f"  Status: {status_color}{result['status']}{Fore.RESET}")
            print(f"  Send SMS: {sms_status_color}{sms_status}{Fore.RESET}")

            if result['times']:
                print(f"  Times: {', '.join([f'{time:.2f}' for time in result['times'] if time is not None])}")

            print(f"  Min: {result['min']:.3f}" if isinstance(result['min'], (int, float)) else 'N/A')
            print(f"  Avg: {result['avg']:.3f}" if isinstance(result['avg'], (int, float)) else 'N/A')
            print(f"  Max: {result['max']:.3f}" if isinstance(result['max'], (int, float)) else 'N/A')
            print(f"  Jitter: {result['jitter']:.3f}" if isinstance(result['jitter'], (int, float)) else 'N/A')
            print(f"  Packet Loss: {result['packet_loss'] if isinstance(result['packet_loss'], (int, float)) else 'N/A'}")
            print(f"  Avg Latency: {result['avg_latency']:.3f}" if isinstance(result['avg_latency'], (int, float)) else 'N/A')
            print("-" * 30)

        if time.time() - last_reset_time >= 10 * 60:
            reset_status()
            last_reset_time = time.time()
        else:
            save_status_to_file()

       # time.sleep(1) 
