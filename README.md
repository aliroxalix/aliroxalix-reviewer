import random
import time
import json
import threading
import csv
from datetime import datetime
from rich.console import Console
from rich.table import Table
from rich.panel import Panel

console = Console()

countries = {
    "Iran": (3, 5),
    "USA": (10, 2),
    "Japan": (15, 4),
    "Germany": (6, 3),
    "Brazil": (4, 10),
    "India": (12, 6),
    "UK": (7, 2),
    "China": (14, 5)
}

attack_types = [
    "DDoS",
    "Brute Force",
    "SQL Injection",
    "Ransomware",
    "Malware",
    "Phishing"
]

log_file = "attacks.json"
stats_file = "stats.json"
csv_file = "attacks.csv"

lock = threading.Lock()

stats = {
    "LOW": 0,
    "MEDIUM": 0,
    "HIGH": 0,
    "CRITICAL": 0,
    "TOTAL": 0,
    "COUNTRIES": {}
}


def load_stats():
    global stats
    try:
        with open(stats_file, "r") as f:
            stats = json.load(f)
    except:
        pass


def save_stats():
    with open(stats_file, "w") as f:
        json.dump(stats, f, indent=4)


def save_attack(attack):
    try:
        with open(log_file, "r") as f:
            data = json.load(f)
    except:
        data = []

    data.append(attack)

    data = data[-200:]  # جلوگیری از سنگین شدن فایل

    with open(log_file, "w") as f:
        json.dump(data, f, indent=4)

    with open(csv_file, "a", newline="") as f:
        writer = csv.writer(f)
        writer.writerow(attack.values())


def draw_map(source, target):
    grid = [["." for _ in range(20)] for _ in range(10)]

    sx, sy = countries[source]
    tx, ty = countries[target]

    grid[sy % 10][sx % 20] = "S"
    grid[ty % 10][tx % 20] = "T"

    return "\n".join([" ".join(row) for row in grid])


def generate_attack():
    global stats

    attack_id = 0

    while True:
        time.sleep(2)

        source = random.choice(list(countries.keys()))
        target = random.choice(list(countries.keys()))
        while source == target:
            target = random.choice(list(countries.keys()))

        attack_type = random.choice(attack_types)
        severity = random.choice(["LOW", "MEDIUM", "HIGH", "CRITICAL"])

        attack_id += 1

        with lock:
            stats["TOTAL"] += 1
            stats[severity] += 1
            stats["COUNTRIES"][source] = stats["COUNTRIES"].get(source, 0) + 1

        attack = {
            "id": attack_id,
            "time": datetime.now().strftime("%H:%M:%S"),
            "type": attack_type,
            "source": source,
            "target": target,
            "severity": severity
        }

        save_attack(attack)
        save_stats()

        render(attack)


def render(attack):
    console.clear()

    color_map = {
        "LOW": "green",
        "MEDIUM": "cyan",
        "HIGH": "yellow",
        "CRITICAL": "red"
    }

    table = Table(title="CYBERNEXUS LIVE DASHBOARD")

    table.add_column("Field")
    table.add_column("Value")

    for k, v in attack.items():
        table.add_row(str(k), str(v))

    console.print(Panel(table, style=color_map[attack["severity"]]))

    console.print("\n[bold magenta]WORLD MAP[/bold magenta]")
    console.print(draw_map(attack["source"], attack["target"]))

    console.print("\n[bold cyan]LIVE STATS[/bold cyan]")
    console.print(stats)

    console.print("\n[bold green]SYSTEM RUNNING...[/bold green]")


def leaderboard():
    sorted_data = sorted(stats.get("COUNTRIES", {}).items(), key=lambda x: x[1], reverse=True)

    table = Table(title="Top Attacking Countries")
    table.add_column("Country")
    table.add_column("Attacks")

    for c, v in sorted_data[:5]:
        table.add_row(c, str(v))

    console.print(table)


if __name__ == "__main__":
    load_stats()

    t = threading.Thread(target=generate_attack)
    t.daemon = True
    t.start()

    while True:
        time.sleep(10)
        leaderboard()
