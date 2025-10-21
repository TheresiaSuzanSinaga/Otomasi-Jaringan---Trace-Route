# Otomasi-Jaringan---Trace-Route
Berisi program yang terkait dengan dengan Trace Route, misalnya saya ingin tau jika terjadi kemacetan pada saat ngeping atau men search sesuatu, dimana titik macetnya itu berada.

# Menyimpan file dengan nama nano network_diagnostic_v2.py

# Mengetik kode programnya
import subprocess
import platform
import re
import time
import statistics

def ping_host(host, count=4):
    """Ping host dan kembalikan rata-rata latency & packet loss."""
    system = platform.system().lower()
    command = ["ping", "-c", str(count), host] if system != "windows" else ["ping", "-n", str(count), host]
    
    try:
        result = subprocess.run(command, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
        output = result.stdout
        
        # Cari packet loss
        packet_loss_match = re.search(r"(\d+)% packet loss", output)
        packet_loss = int(packet_loss_match.group(1)) if packet_loss_match else 0
        
        # Cari latency
        latencies = re.findall(r"time[=<]\s*(\d+\.?\d*)\s*ms", output)
        latencies = [float(x) for x in latencies]
        avg_latency = statistics.mean(latencies) if latencies else None
        
        return avg_latency, packet_loss
    except Exception as e:
        print(f"⚠️ Error saat ping: {e}")
        return None, 100


def traceroute(host):
    """Lakukan traceroute dan tampilkan hasil + analisis."""
    system = platform.system().lower()
    command = ["tracert", "-d", host] if system == "windows" else ["traceroute", "-n", host]
    
    print("\n🔍 Melacak rute jaringan (traceroute)...")
    print("----------------------------------------------------")

    try:
        result = subprocess.run(command, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
        output = result.stdout

        hops = []
        for line in output.splitlines():
            if re.match(r"^\s*\d+", line):
                hops.append(line.strip())

        if not hops:
            print("❌ Tidak ada data traceroute, mungkin tool belum terinstal.")
            return

        high_latency = []
        timeouts = []

        for hop in hops:
            print(hop)
            latencies = re.findall(r"(\d+\.?\d*)\s*ms", hop)
            if latencies:
                avg_latency = sum(map(float, latencies)) / len(latencies)
                if avg_latency > 200:
                    high_latency.append((hop, avg_latency))
            elif "*" in hop:
                timeouts.append(hop)

        print("\n📍 Analisis Rute:")
        if not high_latency and not timeouts:
            print("✅ Tidak ada kemacetan terdeteksi di jalur ini.")
        else:
            if high_latency:
                print("⚠️ Hop dengan latency tinggi (>200 ms):")
                for hop, delay in high_latency:
                    print(f"   - {hop} | Delay rata-rata: {delay:.1f} ms")

            if timeouts:
                print("\n❌ Hop yang tidak merespons (timeout):")
                for hop in timeouts:
                    print(f"   - {hop}")

    except FileNotFoundError:
        print("❗ Tool traceroute/tracert tidak ditemukan.")
        if system == "linux":
            print("   Instal dengan: sudo apt install traceroute")


def monitor_connection(host, interval=5, trace_interval=60):
    """Monitor koneksi dan jalankan traceroute otomatis atau berkala."""
    print(f"\n🚀 Memulai monitoring koneksi ke {host} setiap {interval} detik...")
    print(f"🕒 Traceroute otomatis akan dijalankan setiap {trace_interval} detik.")
    print("Tekan Ctrl + C untuk berhenti.\n")

    last_trace_time = 0

    try:
        while True:
            avg_latency, packet_loss = ping_host(host)
            status = "OK"
            quality = "Bagus"

            if avg_latency is None:
                status = "DOWN"
                quality = "Buruk"
            elif packet_loss > 50:
                quality = "Buruk"
            elif avg_latency > 200:
                quality = "Lambat"

            print(f"[{time.strftime('%H:%M:%S')}] Host: {host} | Latency: {avg_latency} ms | Loss: {packet_loss}% | Status: {status} | Kualitas: {quality}")

            # Jalankan traceroute otomatis kalau koneksi bermasalah
            if status == "DOWN" or quality == "Buruk":
                print("\n🚨 Terdeteksi gangguan jaringan — menjalankan traceroute...\n")
                traceroute(host)
                last_trace_time = time.time()

            # Jalankan traceroute berkala setiap trace_interval detik
            elif time.time() - last_trace_time >= trace_interval:
                print(f"\n🕒 Traceroute berkala (setiap {trace_interval} detik)...\n")
                traceroute(host)
                last_trace_time = time.time()

            time.sleep(interval)

    except KeyboardInterrupt:
        print("\n🛑 Monitoring dihentikan oleh pengguna.")


if __name__ == "__main__":
    host = input("Masukkan alamat host (contoh: google.com): ").strip()
    try:
        interval = int(input("Masukkan interval ping (detik, default 5): ") or "5")
        trace_interval = int(input("Masukkan interval traceroute (detik, default 60): ") or "60")
    except ValueError:
        interval, trace_interval = 5, 60

    monitor_connection(host, interval, trace_interval)

# Kemudian save programnya dengan cara ctrl o, enter, ctrl x

# lalu jalankan dengan cara python3 network_diagnostic_v2.py
# kemungkinan outputnya akan tampak seperti dibawah ini
Masukkan alamat host (contoh: google.com): youtube.com
Masukkan interval ping (detik, default 5): 3
Masukkan interval traceroute (detik, default 60): 60

🚀 Memulai monitoring koneksi ke youtube.com setiap 3 detik...
🕒 Traceroute otomatis akan dijalankan setiap 60 detik.
Tekan Ctrl + C untuk berhenti.

[01:45:08] Host: youtube.com | Latency: 12.4 ms | Loss: 0% | Status: OK | Kualitas: Bagus

🕒 Traceroute berkala (setiap 60 detik)...


🔍 Melacak rute jaringan (traceroute)...
----------------------------------------------------
❌ Tidak ada data traceroute, mungkin tool belum terinstal.
[01:45:14] Host: youtube.com | Latency: 12.2 ms | Loss: 0% | Status: OK | Kualitas: Bagus
[01:45:21] Host: youtube.com | Latency: 12.025 ms | Loss: 0% | Status: OK | Kualitas: Bagus
[01:45:27] Host: youtube.com | Latency: 11.575 ms | Loss: 0% | Status: OK | Kualitas: Bagus
[01:45:33] Host: youtube.com | Latency: 20.7 ms | Loss: 0% | Status: OK | Kualitas: Bagus
[01:45:39] Host: youtube.com | Latency: 12.15 ms | Loss: 0% | Status: OK | Kualitas: Bagus
[01:45:45] Host: youtube.com | Latency: 11.700000000000001 ms | Loss: 0% | Status: OK | Kualitas: Bagus
[01:45:51] Host: youtube.com | Latency: 12.4 ms | Loss: 0% | Status: OK | Kualitas: Bagus
[01:45:57] Host: youtube.com | Latency: 12.6 ms | Loss: 0% | Status: OK | Kualitas: Bagus
[01:46:03] Host: youtube.com | Latency: 12.225 ms | Loss: 0% | Status: OK | Kualitas: Bagus


