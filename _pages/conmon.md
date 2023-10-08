---
title: Continuous Monitoring
permalink: /conmon/
---

This is the latest, most complete version of the scripts for
[Detailed system and process monitoring]({{ site.baseurl }}/2020/03/31/detailed-system-and-process-monitoring.html).

To run in slow systems such as Raspberry Pi computers, including any one from the 
[Zero W (v1)](https://www.raspberrypi.com/products/raspberry-pi-zero-w/)
to the 
[4 model B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/), the **single-thread** script
([`conmon-st`](#conmon-st)) is preferred, with a sampling
delay based on each Raspberry CPU to avoid interfering with
other processes.
To run in faster systems such as servers and desktop PCs,
the **multi-thread** script ([`conmon-mt`](#conmon-mt))
is preferred, with much smaller sampling delays to capture
fast changes in resource utilization more accurately.

In addition to the main `conmon` process, there are a few
scripts that usually only make sense to run on a single host
in the network:

*  [`conmon-speedtest`](#conmon-speedtest) reports the
   download and upload speeds (**Mbps**) and ping (**ms**)
   as reported by
   [sivel/speedtest-cli](https://github.com/sivel/speedtest-cli)
*  [`conmon-mystrom`](#conmon-mystrom) reports the telemetry
   obtained via the **Get report** method in the
   [myStrom REST API](https://api.mystrom.ch/) from one or
   more smart plug/switch devices that report energy usage.

## How to Install

Choose *a* version of the `conmon` script and install it as
`/usr/local/bin/conmon` and run it as a service by creating
`/etc/systemd/system/conmon.service` as follows:

```ini
[Unit]
Description=Continuous Monitoring
After=influxd.service
Wants=influxd.service

[Service]
ExecStart=/usr/local/bin/conmon
Restart=on-failure
StandardOutput=null

[Install]
WantedBy=multi-user.target
```

Then enable and start the services in `systemd`:

```
# systemctl enable conmon.service
# systemctl daemon-reload
# systemctl start conmon.service
# systemctl status conmon.service
```

## Single-thread

### `deploy-to-rpis`

In environments with multiple Raspberry Pi computers, it may
be useful to use this script (`deploy-to-rpis`) to deploy the
latest version of the script to all computers at once.

```bash
#!/bin/bash
#
# Deplay conmon-st to Raspberry Pi hosts.

for host in pi-z1 pi-z2 pi3a pi-f1; do
  if nc 2>&1 -zv ${host} 22 | grep -q succeeded; then
    echo "Deploying to ${host} ..."
    # Copy over the entire repo.
    scp -qr ../conmon pi@${host}:src/ 2>/dev/null
    # Install single-thread script.
    ssh pi@${host} "sudo cp /home/pi/src/conmon/conmon-st /usr/local/bin/conmon" 2>/dev/null
    ssh pi@${host} "sudo systemctl restart conmon.service" 2>/dev/null
  fi
done
```

### `conmon-st`

```bash
#!/bin/bash
#
# Export system monitoring metrics to influxdb.

# InfluxDB target.
DBNAME=monitoring
TARGET='http://lexicon:8086'

# Default delay between each POST request to InfluxDB.
DELAY_POST=4

# Data file for batch POST.
DDIR="/dev/shm/$$"
DATA="${DDIR}/DATA.txt"
mkdir -p "${DDIR}"

host=$(hostname)

timestamp_ns() {
  date +'%s%N'
}

store_line() {
  # Write a line of data to the temporary in-memory file.
  # Exit immediately if this fails.
  echo $1 >>"${DATA}" || exit 1
}

report_top_per_process() {
  # Per-process CPU(%) & MEM(bytes) usage.
  # Re-use the ptop file created by report_top().
  ts=$1
  ptop=$2
  awk '{print $4}' "${ptop}" | sort -u | while read cmd; do
    user=$(grep " ${cmd}\$" ${ptop} | cut -f1 -d' ' | sort -u | head -1)
    # CPU per proccess, only when > 0
    cpu=$(grep " ${cmd}\$" ${ptop} | cut -f2 -d' ' | grep -v '^0\.0$' | tr '\n' '+' | sed 's/+$/\n/' | bc -ql)
    if [[ ! -z "${cpu}" ]]; then
      store_line "top_cpu,host=${host},user=${user},command=${cmd} value=${cpu} ${ts}"
    fi
    # Memory per proccess, only when > 1%
    mem=$(grep " ${cmd}\$" ${ptop} | cut -f3 -d' ' | grep -v '^0\.0$' | tr '\n' '+' | sed 's/+$/\n/' | bc -ql | sed 's/^\./0./')
    # Multiply %mem by total system RAM.
    tot=$(free -b | grep 'Mem:' | awk '{print $2}')
    ram=$(bc -ql <<<"${tot} * ${mem} / 100")
    if [[ ! -z "${ram}" ]]; then
      store_line "top_mem,host=${host},user=${user},command=${cmd} value=${ram} ${ts}"
    fi
  done
  rm -f "${ptop}"
}

report_top() {
  # Stats from top: CPU (overall and per process) and RAM (per process).
  # Depends on: top, free.
  top_cmd=$(command -v top)
  top="${DDIR}/top"
  ts=$(timestamp_ns)
  ${top_cmd} -b -c -n 1 -w 512 |
    grep -vE ' 0\..   0\..|^top|^Tasks|^%|^MiB|^$|[[:blank:]]*PID USER' |
    awk '{print $2,$9,$10,$12,$13,$14,$15,$16,$17,$18,$19,$20}' |
    tr '\\' '/' |
    sed 's/\.minecraft\/bin\/[0-9a-f]\+.*/minecraft/' |
    sed 's/[C-Z]:\/.*\/\([a-zA-Z0-9 _-]\+\.[a-z][a-z][a-z]\).*/\1/' |
    sed 's/\/[^ ]*\///' | sed 's/\(bash\|sh\|python\|python3\) .*\///' |
    tr -d '[' |
    tr -d ']' |
    awk '{print $1,$2,$3,$4}' >"${top}"
  # Total CPU(%) usage.
  cpu_load=$(awk '{print $2}' ${top} | grep -v '^0\.0$' | tr '\n' '+' | sed 's/+$/\n/' | bc -ql)
  store_line "top,host=${host} value=${cpu_load} ${ts}"
  # Launch the slower per-process metrics in the background.
  ptop="${top}.$RANDOM"
  mv "${top}" "${ptop}"
  report_top_per_process "${ts}" "${ptop}" &
}

report_vcgencmd() {
  # Raspberry Pi CPU temperature.
  # Depends on: vcgencmd.
  vcgencmd=$(command -v vcgencmd)
  if [ -z "${vcgencmd}" ] || [ ! -f "${vcgencmd}" ]; then
    return
  fi
  ts=$(timestamp_ns)
  cpu_temp=$(${vcgencmd} measure_temp | cut -f2 -d= | cut -f1 -d"'")
  store_line "vcgencmd,metric=temp,host=${host} value=${cpu_temp} ${ts}"
  if grep -q 'Pi 4' /proc/device-tree/model; then
    ts=$(timestamp_ns)
    pmic_temp=$(${vcgencmd} measure_temp pmic | cut -f2 -d= | cut -f1 -d"'")
    store_line "vcgencmd,metric=temp_pmic,host=${host} value=${pmic_temp} ${ts}"
  fi
  cat "${DATA}" | cut -f1 -d, | sort | uniq -c
}

report_sensors() {
  # CPU, SSD, NVMe temperatures and other sensors (if available).
  # Depends on: sensors.
  sensors=$(command -v sensors)
  if [ -z "${sensors}" ] || [ ! -f "${sensors}" ]; then
    return
  fi
  sensors_u="${DDIR}/sensors"
  ts=$(timestamp_ns)
  "${sensors}" -u >"${sensors_u}"
  grep '^[^ -].*-.*' "${sensors_u}" | while read adapter; do
    sensors_ua="${DDIR}/sensors_${adapter}"
    # This is tricky: -A18 is just enough to capture SSD temps without causing
    # metrics spilling too much across adapters.
    grep -E "$adapter|_input|:\$" "${sensors_u}" | grep -vE ': 0\.000|: -[0-9]+\.0' | grep -EB1 "fan._input|temp._input|${adapter}" | grep -A18 "${adapter}" >"${sensors_ua}"
    grep '^[^ -]' "${sensors_ua}" | grep -v "${adapter}" | cut -f1 -d: | while read name; do
      value=$(grep -A2 "${name}" ${sensors_ua} | grep '_input' | head -1 | cut -f2 -d: | cut -f2 -d' ' | sed 's/\.000$//')
      store_line "sensors,host=${host},adapter=${adapter},name=${name/ /_} value=${value} ${ts}"
    done
  done
}

report_intel_gpu_top() {
  # Intel GPU.
  # Depends on: intel_gpu_top (as root).
  intel_gpu_top=$(command -v intel_gpu_top)
  if [ -z $intel_gpu_top ] || [ ! -f $intel_gpu_top ]; then
    return
  fi
  ts=$(timestamp_ns)
  # TODO: try with -J and find the most meaningful metrics.
  gpu_load=$(sudo ${intel_gpu_top} -o - | head -3 | tail -1 | awk '{print $2}')
  store_line "intel_gpu,host=${host} value=${value} ${ts}"
}

report_nvidia_smi() {
  # NVidia GPU.
  # Depends on: nvidia-smi
  nvidia_smi=$(command -v nvidia-smi)
  if [ -z "${nvidia_smi}" ] || [ ! -f "${nvidia_smi}" ]; then
    return
  fi
  ts=$(timestamp_ns)
  temp=$(${nvidia_smi} -i 0 --query-gpu=temperature.gpu --format=csv,noheader)
  util=$(${nvidia_smi} -i 0 --query-gpu=utilization.gpu --format=csv,noheader | cut -f1 -d' ')
  vram=$(${nvidia_smi} -i 0 --query-gpu=memory.used --format=csv,noheader | cut -f1 -d' ')
  draw=$(${nvidia_smi} -i 0 --query-gpu=power.draw --format=csv,noheader | cut -f1 -d' ')
  fans=$(${nvidia_smi} -i 0 --query-gpu=fan.speed --format=csv,noheader | cut -f1 -d' ')
  store_line "nvidia_smi,host=${host},metric=temperature value=${temp} ${ts}"
  store_line "nvidia_smi,host=${host},metric=utilization value=${util} ${ts}"
  store_line "nvidia_smi,host=${host},metric=memory value=${vram} ${ts}"
  store_line "nvidia_smi,host=${host},metric=power value=${draw} ${ts}"
  store_line "nvidia_smi,host=${host},metric=fan value=${fans} ${ts}"
}

report_free() {
  # Stats from free: RAM used, buffered/cached, free.
  # Depends on: free.
  ts=$(timestamp_ns)
  mem_used=$(free -b | grep 'Mem:' | awk '{print $3}')
  mem_free=$(free -b | grep 'Mem:' | awk '{print $4}')
  mem_buff=$(free -b | grep 'Mem:' | awk '{print $6}')
  store_line "free,host=${host},metric=used value=${mem_used} ${ts}"
  store_line "free,host=${host},metric=free value=${mem_free} ${ts}"
  store_line "free,host=${host},metric=buff_cache value=${mem_buff} ${ts}"
}

report_df() {
  # Stats from df: used/free space per file system.
  # Depends on: df.
  ts=$(timestamp_ns)
  df -k | egrep -v 'udev|loop|/sys|/run|/dev$' |
    grep ' /' | awk '{print $4,$5,$6}' |
    while read line; do
      # Note free space is given in 1KB blocks.
      fs_path=$(echo "${line}" | cut -f3 -d' ')
      fs_used=$(echo "${line}" | cut -f2 -d' ' | cut -f1 -d'%')
      fs_free=$(echo "${line}" | cut -f1 -d' ')
      fs_free=$((1024 * fs_free))
      store_line "df,host=${host},path=${fs_path},metric=tot_free value=${fs_free} ${ts}"
      store_line "df,host=${host},path=${fs_path},metric=pct_used value=${fs_used} ${ts}"
    done
}

report_diskstats() {
  # Disk I/O stats from /proc/diskstats.
  # Depends on: /proc/diskstats.
  ts=$(timestamp_ns)
  # The /proc/diskstats file displays the I/O statistics of block devices.
  # Each line contains the following fields (and more, omitted here):
  #  1  major number
  #  2  minor mumber
  #  3  device name
  #  4  reads completed successfully
  #  5  reads merged
  #  6  sectors read
  #  7  time spent reading (ms)
  #  8  writes completed
  #  9  writes merged
  # 10  sectors written
  # 11  time spent writing (ms)
  # Source: https://www.kernel.org/doc/Documentation/ABI/testing/procfs-diskstats
  grep -v ' 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0' /proc/diskstats | egrep -v 'loop|[hs]d[a-z][1-9]|k0p[0-9]' | awk '{print $3,$4,$6,$7,$8,$10,$11}' | sort >$DDIR/disk1
  if [[ -f "${DDIR}/disk0" ]]; then
    # Columns used: disk reads rsect rtime writs wsect wtime
    # disk:   3  device name
    # reads:  4  reads completed successfully
    # rsect:  6  sectors read
    # rtime:  7  time spent reading (ms)
    # writs:  8  writes completed
    # wsect: 10  sectors written
    # wtime: 11  time spent writing (ms)
    paste "${DDIR}/disk0" "${DDIR}/disk1" |
      while read \
        disk0 reads0 rsect0 rtime0 writs0 wsect0 wtime0 \
        disk1 reads1 rsect1 rtime1 writs1 wsect1 wtime1; do
        # Skip momentary inconsistencies when disks are added or removed.
        if [ "${disk0}" == "${disk1}" ]; then
          disk=$disk0
          # Assume 512-byte sectors.
          rsect=$((512 * (rsect1 - rsect0)))
          wsect=$((512 * (wsect1 - wsect0)))
          store_line "diskstats,host=${host},device=${disk},metric=bytes_read    value=${rsect} ${ts}"
          store_line "diskstats,host=${host},device=${disk},metric=bytes_written value=${wsect} ${ts}"
          # Successful I/O ops.
          reads=$((reads1 - reads0))
          writs=$((writs1 - writs0))
          store_line "diskstats,host=${host},device=${disk},metric=successful_reads  value=${reads} ${ts}"
          store_line "diskstats,host=${host},device=${disk},metric=successful_writes value=${writs} ${ts}"
          # Time spent.
          read_ms=$((rtime1 - rtime0))
          writ_ms=$((wtime1 - wtime0))
          store_line "diskstats,host=${host},device=${disk},metric=time_spent_reads  value=${read_ms} ${ts}"
          store_line "diskstats,host=${host},device=${disk},metric=time_spent_writes value=${writ_ms} ${ts}"
        fi
      done
  fi
  mv "${DDIR}/disk1" "${DDIR}/disk0"
}

bytes_from_value_and_units() {
  value=$1
  units=$2
  factor=1
  case "${units}" in
  "K/s")
    factor=1024
    ;;
  "M/s")
    factor=1024*1024
    ;;
  "G/s")
    factor=1024*1024*1024
    ;;
  "T/s")
    factor=1024*1024*1024*1024
    ;;
  esac
  bc -ql <<<"${factor} * ${value}"
}

report_iotop() {
  # I/O stats from iotop (per process).
  # Depends on: iotop (as root).
  iotop=$(command -v iotop)
  if [ -z $iotop ] || [ ! -f $iotop ]; then
    return
  fi
  ts=$(timestamp_ns)
  iotop_stats=$DDIR/iotop
  # The Linux kernel interfaces that iotop relies on now require root
  # privileges or the NET_ADMIN capability. This change occurred because a
  # security issue (CVE-2011-2494) was found that allows leakage of sensitive
  # data across user boundaries. If you require the ability to run iotop as a
  # non-root user, please configure sudo to allow you to run iotop as root.
  sudo $iotop -b -P -n 1 |
    grep -Ev 'task_delayacct|locale|DISK READ|0.00 B/s    0.00 B/s' |
    tr -d '[' |
    tr -d ']' |
    sed 's/kworker\///' |
    sed 's/u64:[0-9]\+-//' |
    awk '{print $4,$5,$6,$7,$9}' \
      >"${iotop_stats}"
  # Return if iotop could not be run successfully.
  # This is typically caused by 'Netlink error: Operation not permitted'.
  if [[ $? -gt 0 ]]; then
    return
  fi
  # Report I/O stats for each process.
  # Note: this aggregates I/O per process name (basename), across all PIDs.
  cat "${iotop_stats}" |
    cut -f5 -d' ' |
    sort -u |
    while read -r process; do
      process_read=0
      process_write=0
      while read -r line; do
        read_value=$(echo "${line}" | cut -f1 -d' ')
        read_units=$(echo "${line}" | cut -f2 -d' ')
        read_bytes=$(bytes_from_value_and_units "${read_value}" "${read_units}")
        process_read=$(bc -ql <<<"${process_read}+${read_bytes}")
        write_value=$(echo "${line}" | cut -f3 -d' ')
        write_units=$(echo "${line}" | cut -f4 -d' ')
        write_bytes=$(bytes_from_value_and_units "${write_value}" "${write_units}")
        process_write=$(bc -ql <<<"${process_write}+${write_bytes}")
      done < <(grep " ${process}\$" "${iotop_stats}")
      if [[ $(echo "${process_read}" | sed 's/\..*//') -gt 0 ]]; then
        store_line "iotop,host=${host},command=${process},metric=read value=${process_read} ${ts}"
      fi
      if [[ "$(echo "${process_write}" | sed 's/\..*//')" -gt 0 ]]; then
        store_line "iotop,host=${host},command=${process},metric=write value=${process_write} ${ts}"
      fi
    done
}

report_net_dev() {
  # Network I/O stats from /proc/net/dev.
  # Depends on: /proc/net/dev.
  ts=$(timestamp_ns)
  grep -Ev 'Inter|face' /proc/net/dev | tr -d ':' | awk '{print $1,$2,$10}' | sort >"${DDIR}/net1"
  if [[ -f "${DDIR}/net0" ]]; then
    ts0=$(cat "${DDIR}/net0-ts")
    echo "${ts}" >"${DDIR}/net1-ts"
    paste "${DDIR}/net0" "${DDIR}/net1" | while read -r dev0 rx0 tx0 dev1 rx1 tx1; do
      # Skip momentary inconsistencies when devs are added or removed.
      if [[ "${dev0}" == "${dev1}" ]]; then
        # Compute rx/tx bytes / sec (ts is in nanoseconds).
        rx=$(bc -ql <<<"1000000000 * (${rx1} - ${rx0}) / (${ts} - ${ts0})")
        tx=$(bc -ql <<<"1000000000 * (${tx1} - ${tx0}) / (${ts} - ${ts0})")
        store_line "net_dev,host=${host},device=${dev0},metric=rx value=${rx} ${ts}"
        store_line "net_dev,host=${host},device=${dev0},metric=tx value=${tx} ${ts}"
      fi
    done
  fi
  mv "${DDIR}/net1" "${DDIR}/net0"
  mv "${DDIR}/net1-ts" "${DDIR}/net0-ts"
}

report_du() {
  # Disk usage stats from du.
  # Files and directories to monitor in /etc/conmon/du
  # Depends on du (as root).
  if [ -z /etc/conmon/du ] || [ ! -f $/etc/conmon/du ]; then
    return
  fi
  ts=$(timestamp_ns)
  while read -r path </etc/conmon/du; do
    kbytes=$(sudo du -s "${path}" | awk '{print $1}')
    bytes=$((1024 * kbytes))
    store_line "du,host=${host},path=${path} value=${bytes} ${ts}"
  done
}

post_lines_to_influxdb() {
  # POST data to InfluxDB in batch, when target is available.
  # Depends on: nc.
  sleep ${DELAY_POST}
  host_and_port=$(echo "${TARGET}" | sed 's/.*\///' | tr : ' ')
  if nc 2>&1 -zv ${host_and_port} | grep -q succeeded; then
    # All other tasks write data to the file in append mode (>>).
    # This task reads everything at once and immediately deletes the file.
    # This makes all the other tasks write to the same file, created anew.
    mv -f "${DATA}" "${DATA}.POST"
    cut -f1 -d, "${DATA}.POST" | sort | uniq -c
    curl >/dev/null 2>/dev/null -i -XPOST "${TARGET}/write?db=${DBNAME}" --data-binary @"${DATA}.POST"
  fi
}

pause_depending_on_rpi_model() {
  # Pause for a few seconds depending on which model of Raspberry Pi.
  # The delay depends also on the system load as reported by top.
  # Depends on: grep, /proc/cpu_info.
  top_cmd=$(command -v top)
  load=$(${top_cmd} -b -n 1 | head -1 | sed 's/.*load average: //' | cut -f1 -d, | tr -d '.' | sed 's/^0//')
  declare -A cpu_mhz_per_model=(
    ["Raspberry_Pi_Zero_W_Rev_1_1"]=1000         # 1x1000 MHz
    ["Raspberry_Pi_Zero_2_W_Rev_1_0"]=4000       # 4x1000 MHz
    ["Raspberry_Pi_3_Model_A_Plus_Rev_1_0"]=1400 # 1x1400 MHz
    ["Raspberry_Pi_3_Model_B_Rev_1_3"]=4800      # 4x1200 MHz
    ["Raspberry_Pi_4_Model_B_Rev_1_4"]=6000      # 4x1500 MHz
  )
  model=$(grep Model /proc/cpuinfo | sed 's/.*: //' | tr ' ' '_' | tr '.' '_')
  mhz=${cpu_mhz_per_model["${model}"]}
  delay=$((250 * load / mhz))
  sleep ${delay}
}

# Run all the above tasks in a loop.
# Each task is responsible of its own checks.
while true; do
  report_df
  report_du
  report_top
  report_free
  report_iotop
  report_sensors
  report_net_dev
  report_vcgencmd
  report_diskstats
  report_nvidia_smi
  report_intel_gpu_top
  post_lines_to_influxdb
  pause_depending_on_rpi_model
done
```

## Single-thread

To run in slow systems such as Raspberry Pi computers, including any one from the 
[Zero W (v1)](https://www.raspberrypi.com/products/raspberry-pi-zero-w/)
to the 
[4 model B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/), the single-thread version below
([`conmon-st`](#conmon-st)) is preferred, with a sampling
delay based on each Raspberry CPU to avoid interfering with
other processes.

### `deploy-to-pcs`

In environments with multiple (desktop / server) computers,
it may be useful to use this script (`deploy-to-pcs`) to
deploy the latest version of the script to all computers.

```bash
#!/bin/bash
#
# Deplay conmon-mt to PC hosts.

for host in computer smart-computer lexicon rapture; do
  if nc 2>&1 -zv ${host} 22 | grep -q succeeded; then
    echo "Deploying to ${host} ..."
    # Copy over the entire repo.
    scp -qr ../conmon root@${host}: 2>/dev/null
    # Install single-thread script.
    ssh root@${host} "cp /root/conmon/conmon-mt /usr/local/bin/conmon" 2>/dev/null
    ssh root@${host} "systemctl restart conmon.service" 2>/dev/null
  fi
done
```

### `conmon-mt`

```bash
#!/bin/bash
#
# Export system monitoring metrics to influxdb.

# InfluxDB target.
DBNAME=monitoring
TARGET='http://lexicon:8086'

# Default delay between each POST request to InfluxDB.
DELAY_POST=4

# Default delay between each round of "fast" metrics.
DELAY_FAST=2

# Default delay between each round of top (neither fast nor slow).
DELAY_TOP=2

# Default delay between each round of slow metrics (e.g. I/O bound).
DELAY_SLOW=300

# Data file for batch POST.
DDIR="/dev/shm/$$"
DATA="${DDIR}/DATA.txt"
mkdir -p "${DDIR}"

host=$(hostname)

timestamp_ns() {
  date +'%s%N'
}

store_line() {
  # Write a line of data to the temporary in-memory file.
  # Exit immediately if this fails.
  echo $1 >>"${DATA}" || exit 1
}

report_top_per_process() {
  # Per-process CPU(%) & MEM(bytes) usage.
  # Re-use the ptop file created by report_top().
  ts=$1
  ptop=$2
  awk '{print $4}' "${ptop}" | sort -u | while read cmd; do
    user=$(grep " ${cmd}\$" ${ptop} | cut -f1 -d' ' | sort -u | head -1)
    # CPU per proccess, only when > 0
    cpu=$(grep " ${cmd}\$" ${ptop} | cut -f2 -d' ' | grep -v '^0\.0$' | tr '\n' '+' | sed 's/+$/\n/' | bc -ql)
    if [[ ! -z "${cpu}" ]]; then
      store_line "top_cpu,host=${host},user=${user},command=${cmd} value=${cpu} ${ts}"
    fi
    # Memory per proccess, only when > 1%
    mem=$(grep " ${cmd}\$" ${ptop} | cut -f3 -d' ' | grep -v '^0\.0$' | tr '\n' '+' | sed 's/+$/\n/' | bc -ql | sed 's/^\./0./')
    # Multiply %mem by total system RAM.
    tot=$(free -b | grep 'Mem:' | awk '{print $2}')
    ram=$(bc -ql <<<"${tot} * ${mem} / 100")
    if [[ ! -z "${ram}" ]]; then
      store_line "top_mem,host=${host},user=${user},command=${cmd} value=${ram} ${ts}"
    fi
  done
  rm -f "${ptop}"
}

report_top() {
  # Stats from top: CPU (overall and per process) and RAM (per process).
  # Depends on: top, free.
  while true; do
    top_cmd=$(command -v top)
    top="${DDIR}/top"
    ts=$(timestamp_ns)
    ${top_cmd} -b -c -n 1 -w 512 |
      grep -vE ' 0\..   0\..|^top|^Tasks|^%|^MiB|^$|[[:blank:]]*PID USER' |
      awk '{print $2,$9,$10,$12,$13,$14,$15,$16,$17,$18,$19,$20}' |
      tr '\\' '/' |
      sed 's/\.minecraft\/bin\/[0-9a-f]\+.*/minecraft/' |
      sed 's/[C-Z]:\/.*\/\([a-zA-Z0-9 _-]\+\.[a-z][a-z][a-z]\).*/\1/' |
      sed 's/\/[^ ]*\///' | sed 's/\(bash\|sh\|python\|python3\) .*\///' |
      tr -d '[' |
      tr -d ']' |
      awk '{print $1,$2,$3,$4}' >"${top}"
    # Total CPU(%) usage.
    cpu_load=$(awk '{print $2}' ${top} | grep -v '^0\.0$' | tr '\n' '+' | sed 's/+$/\n/' | bc -ql)
    store_line "top,host=${host} value=${cpu_load} ${ts}"
    # Launch the slower per-process metrics in the background.
    ptop="${top}.$RANDOM"
    mv "${top}" "${ptop}"
    report_top_per_process "${ts}" "${ptop}" &
    sleep ${DELAY_TOP}
  done
}

report_vcgencmd() {
  # Raspberry Pi CPU temperature.
  # Depends on: vcgencmd.
  vcgencmd=$(command -v vcgencmd)
  if [ -z "${vcgencmd}" ] || [ ! -f "${vcgencmd}" ]; then
    return
  fi
  while true; do
    ts=$(timestamp_ns)
    cpu_temp=$(${vcgencmd} measure_temp | cut -f2 -d= | cut -f1 -d"'")
    store_line "vcgencmd,metric=temp,host=${host} value=${cpu_temp} ${ts}"
    if grep -q 'Pi 4' /proc/device-tree/model; then
      ts=$(timestamp_ns)
      pmic_temp=$(${vcgencmd} measure_temp pmic | cut -f2 -d= | cut -f1 -d"'")
      store_line "vcgencmd,metric=temp_pmic,host=${host} value=${pmic_temp} ${ts}"
    fi
    cat "${DATA}" | cut -f1 -d, | sort | uniq -c
    sleep ${DELAY_FAST}
  done
}

report_sensors() {
  # CPU, SSD, NVMe temperatures and other sensors (if available).
  # Depends on: sensors.
  sensors=$(command -v sensors)
  if [ -z "${sensors}" ] || [ ! -f "${sensors}" ]; then
    return
  fi
  sensors_u="${DDIR}/sensors"
  while true; do
    ts=$(timestamp_ns)
    "${sensors}" -u >"${sensors_u}"
    grep '^[^ -].*-.*' "${sensors_u}" | while read adapter; do
      sensors_ua="${DDIR}/sensors_${adapter}"
      # This is tricky: -A18 is just enough to capture SSD temps without causing
      # metrics spilling too much across adapters.
      grep -E "$adapter|_input|:\$" "${sensors_u}" | grep -vE ': 0\.000|: -[0-9]+\.0' | grep -EB1 "fan._input|temp._input|${adapter}" | grep -A18 "${adapter}" >"${sensors_ua}"
      grep '^[^ -]' "${sensors_ua}" | grep -v "${adapter}" | cut -f1 -d: | while read name; do
        value=$(grep -A2 "${name}" ${sensors_ua} | grep '_input' | head -1 | cut -f2 -d: | cut -f2 -d' ' | sed 's/\.000$//')
        store_line "sensors,host=${host},adapter=${adapter},name=${name/ /_} value=${value} ${ts}"
      done
    done
    sleep ${DELAY_FAST}
  done
}

report_intel_gpu_top() {
  # Intel GPU.
  # Depends on: intel_gpu_top (as root).
  intel_gpu_top=$(command -v intel_gpu_top)
  if [ -z $intel_gpu_top ] || [ ! -f $intel_gpu_top ]; then
    return
  fi
  while true; do
    ts=$(timestamp_ns)
    # TODO: try with -J and find the most meaningful metrics.
    gpu_load=$(sudo ${intel_gpu_top} -o - | head -3 | tail -1 | awk '{print $2}')
    store_line "intel_gpu,host=${host} value=${value} ${ts}"
    sleep ${DELAY_FAST}
  done
}

report_nvidia_smi() {
  # NVidia GPU.
  # Depends on: nvidia-smi
  nvidia_smi=$(command -v nvidia-smi)
  if [ -z "${nvidia_smi}" ] || [ ! -f "${nvidia_smi}" ]; then
    return
  fi
  while true; do
    ts=$(timestamp_ns)
    temp=$(${nvidia_smi} -i 0 --query-gpu=temperature.gpu --format=csv,noheader)
    util=$(${nvidia_smi} -i 0 --query-gpu=utilization.gpu --format=csv,noheader | cut -f1 -d' ')
    vram=$(${nvidia_smi} -i 0 --query-gpu=memory.used --format=csv,noheader | cut -f1 -d' ')
    draw=$(${nvidia_smi} -i 0 --query-gpu=power.draw --format=csv,noheader | cut -f1 -d' ')
    fans=$(${nvidia_smi} -i 0 --query-gpu=fan.speed --format=csv,noheader | cut -f1 -d' ')
    store_line "nvidia_smi,host=${host},metric=temperature value=${temp} ${ts}"
    store_line "nvidia_smi,host=${host},metric=utilization value=${util} ${ts}"
    store_line "nvidia_smi,host=${host},metric=memory value=${vram} ${ts}"
    store_line "nvidia_smi,host=${host},metric=power value=${draw} ${ts}"
    store_line "nvidia_smi,host=${host},metric=fan value=${fans} ${ts}"
    sleep ${DELAY_FAST}
  done
}

report_free() {
  # Stats from free: RAM used, buffered/cached, free.
  # Depends on: free.
  while true; do
    ts=$(timestamp_ns)
    mem_used=$(free -b | grep 'Mem:' | awk '{print $3}')
    mem_free=$(free -b | grep 'Mem:' | awk '{print $4}')
    mem_buff=$(free -b | grep 'Mem:' | awk '{print $6}')
    store_line "free,host=${host},metric=used value=${mem_used} ${ts}"
    store_line "free,host=${host},metric=free value=${mem_free} ${ts}"
    store_line "free,host=${host},metric=buff_cache value=${mem_buff} ${ts}"
    sleep ${DELAY_FAST}
  done
}

report_df() {
  # Stats from df: used/free space per file system.
  # Depends on: df.
  while true; do
    ts=$(timestamp_ns)
    df -k | egrep -v '/run|/pods|udev|loop|/sys|/run|/dev$' |
      grep ' /' | awk '{print $4,$5,$6}' |
      while read line; do
        # Note free space is given in 1KB blocks.
        fs_path=$(echo "${line}" | cut -f3 -d' ')
        fs_used=$(echo "${line}" | cut -f2 -d' ' | cut -f1 -d'%')
        fs_free=$(echo "${line}" | cut -f1 -d' ')
        fs_free=$((1024 * fs_free))
        store_line "df,host=${host},path=${fs_path},metric=tot_free value=${fs_free} ${ts}"
        store_line "df,host=${host},path=${fs_path},metric=pct_used value=${fs_used} ${ts}"
      done
    sleep ${DELAY_FAST}
  done
}

report_diskstats() {
  # Disk I/O stats from /proc/diskstats.
  # Depends on: /proc/diskstats.
  while true; do
    ts=$(timestamp_ns)
    # The /proc/diskstats file displays the I/O statistics of block devices.
    # Each line contains the following fields (and more, omitted here):
    #  1  major number
    #  2  minor mumber
    #  3  device name
    #  4  reads completed successfully
    #  5  reads merged
    #  6  sectors read
    #  7  time spent reading (ms)
    #  8  writes completed
    #  9  writes merged
    # 10  sectors written
    # 11  time spent writing (ms)
    # Source: https://www.kernel.org/doc/Documentation/ABI/testing/procfs-diskstats
    grep -v ' 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0' /proc/diskstats | egrep -v 'loop|[hs]d[a-z][1-9]|k0p[0-9]' | awk '{print $3,$4,$6,$7,$8,$10,$11}' | sort >$DDIR/disk1
    if [[ -f "${DDIR}/disk0" ]]; then
      # Columns used: disk reads rsect rtime writs wsect wtime
      # disk:   3  device name
      # reads:  4  reads completed successfully
      # rsect:  6  sectors read
      # rtime:  7  time spent reading (ms)
      # writs:  8  writes completed
      # wsect: 10  sectors written
      # wtime: 11  time spent writing (ms)
      paste "${DDIR}/disk0" "${DDIR}/disk1" |
        while read \
          disk0 reads0 rsect0 rtime0 writs0 wsect0 wtime0 \
          disk1 reads1 rsect1 rtime1 writs1 wsect1 wtime1; do
          # Skip momentary inconsistencies when disks are added or removed.
          if [ "${disk0}" == "${disk1}" ]; then
            disk=$disk0
            # Assume 512-byte sectors.
            rsect=$((512 * (rsect1 - rsect0)))
            wsect=$((512 * (wsect1 - wsect0)))
            store_line "diskstats,host=${host},device=${disk},metric=bytes_read    value=${rsect} ${ts}"
            store_line "diskstats,host=${host},device=${disk},metric=bytes_written value=${wsect} ${ts}"
            # Successful I/O ops.
            reads=$((reads1 - reads0))
            writs=$((writs1 - writs0))
            store_line "diskstats,host=${host},device=${disk},metric=successful_reads  value=${reads} ${ts}"
            store_line "diskstats,host=${host},device=${disk},metric=successful_writes value=${writs} ${ts}"
            # Time spent.
            read_ms=$((rtime1 - rtime0))
            writ_ms=$((wtime1 - wtime0))
            store_line "diskstats,host=${host},device=${disk},metric=time_spent_reads  value=${read_ms} ${ts}"
            store_line "diskstats,host=${host},device=${disk},metric=time_spent_writes value=${writ_ms} ${ts}"
          fi
        done
    fi
    mv "${DDIR}/disk1" "${DDIR}/disk0"
    sleep ${DELAY_FAST}
  done
}

bytes_from_value_and_units() {
  value=$1
  units=$2
  factor=1
  case "${units}" in
  "K/s")
    factor=1024
    ;;
  "M/s")
    factor=1024*1024
    ;;
  "G/s")
    factor=1024*1024*1024
    ;;
  "T/s")
    factor=1024*1024*1024*1024
    ;;
  esac
  bc -ql <<<"${factor} * ${value}"
}

report_iotop() {
  # I/O stats from iotop (per process).
  # Depends on: iotop (as root).
  iotop=$(command -v iotop)
  if [ -z $iotop ] || [ ! -f $iotop ]; then
    return
  fi
  while true; do
    ts=$(timestamp_ns)
    iotop_stats=$DDIR/iotop
    # The Linux kernel interfaces that iotop relies on now require root
    # privileges or the NET_ADMIN capability. This change occurred because a
    # security issue (CVE-2011-2494) was found that allows leakage of sensitive
    # data across user boundaries. If you require the ability to run iotop as a
    # non-root user, please configure sudo to allow you to run iotop as root.
    sudo $iotop -b -P -n 1 |
      grep -Ev 'task_delayacct|locale|DISK READ|0.00 B/s    0.00 B/s' |
      tr -d '[' |
      tr -d ']' |
      sed 's/kworker\///' |
      sed 's/u64:[0-9]\+-//' |
      awk '{print $4,$5,$6,$7,$9}' \
        >"${iotop_stats}"
    # Return if iotop could not be run successfully.
    # This is typically caused by 'Netlink error: Operation not permitted'.
    if [[ $? -gt 0 ]]; then
      return
    fi
    # Report I/O stats for each process.
    # Note: this aggregates I/O per process name (basename), across all PIDs.
    cat "${iotop_stats}" |
      cut -f5 -d' ' |
      sort -u |
      while read -r process; do
        process_read=0
        process_write=0
        while read -r line; do
          read_value=$(echo "${line}" | cut -f1 -d' ')
          read_units=$(echo "${line}" | cut -f2 -d' ')
          read_bytes=$(bytes_from_value_and_units "${read_value}" "${read_units}")
          process_read=$(bc -ql <<<"${process_read}+${read_bytes}")
          write_value=$(echo "${line}" | cut -f3 -d' ')
          write_units=$(echo "${line}" | cut -f4 -d' ')
          write_bytes=$(bytes_from_value_and_units "${write_value}" "${write_units}")
          process_write=$(bc -ql <<<"${process_write}+${write_bytes}")
        done < <(grep " ${process}\$" "${iotop_stats}")
        if [[ $(echo "${process_read}" | sed 's/\..*//') -gt 0 ]]; then
          store_line "iotop,host=${host},command=${process},metric=read value=${process_read} ${ts}"
        fi
        if [[ "$(echo "${process_write}" | sed 's/\..*//')" -gt 0 ]]; then
          store_line "iotop,host=${host},command=${process},metric=write value=${process_write} ${ts}"
        fi
      done
    sleep ${DELAY_FAST}
  done
}

report_net_dev() {
  # Network I/O stats from /proc/net/dev.
  # Depends on: /proc/net/dev.
  while true; do
    ts=$(timestamp_ns)
    grep -Ev 'Inter|face' /proc/net/dev | tr -d ':' | awk '{print $1,$2,$10}' | sort >"${DDIR}/net1"
    if [[ -f "${DDIR}/net0" ]]; then
      ts0=$(cat "${DDIR}/net0-ts")
      echo "${ts}" >"${DDIR}/net1-ts"
      paste "${DDIR}/net0" "${DDIR}/net1" | while read -r dev0 rx0 tx0 dev1 rx1 tx1; do
        # Skip momentary inconsistencies when devs are added or removed.
        if [[ "${dev0}" == "${dev1}" ]]; then
          # Compute rx/tx bytes / sec (ts is in nanoseconds).
          rx=$(bc -ql <<<"1000000000 * (${rx1} - ${rx0}) / (${ts} - ${ts0})")
          tx=$(bc -ql <<<"1000000000 * (${tx1} - ${tx0}) / (${ts} - ${ts0})")
          store_line "net_dev,host=${host},device=${dev0},metric=rx value=${rx} ${ts}"
          store_line "net_dev,host=${host},device=${dev0},metric=tx value=${tx} ${ts}"
        fi
      done
    fi
    mv "${DDIR}/net1" "${DDIR}/net0"
    mv "${DDIR}/net1-ts" "${DDIR}/net0-ts"
    sleep ${DELAY_FAST}
  done
}

report_du() {
  # Disk usage stats from du.
  # Files and directories to monitor in /etc/conmon/du
  # Depends on du (as root).
  if [ -z /etc/conmon/du ] || [ ! -f $/etc/conmon/du ]; then
    return
  fi
  while true; do
    ts=$(timestamp_ns)
    while read -r path </etc/conmon/du; do
      kbytes=$(sudo du -s "${path}" | awk '{print $1}')
      bytes=$((1024 * kbytes))
      store_line "du,host=${host},path=${path} value=${bytes} ${ts}"
    done
    sleep ${DELAY_SLOW}
  done
}

post_lines_to_influxdb() {
  # POST data to InfluxDB in batch, when target is available.
  # Depends on: curl, nc.
  while true; do
    sleep ${DELAY_POST}
    host_and_port=$(echo $TARGET | sed 's/.*\///' | tr : ' ')
    if nc 2>&1 -zv $host_and_port | grep -q succeeded; then
      # All other tasks write data to the file in append mode (>>).
      # This task reads everything at once and immediately deletes the file.
      # This makes all the other tasks write to the same file, created anew.
      # To avoid losing data between reading and delete the file, rename it,
      # wait a little (DELAY_FAST) for the writes of tasks that already had
      # it open, then other tasks will have to write to a new file.
      mv -f "${DATA}" "${DATA}.POST"
      cut -f1 -d, "${DATA}.POST" | sort | uniq -c
      curl >/dev/null 2>/dev/null -i -XPOST "${TARGET}/write?db=${DBNAME}" --data-binary @"${DATA}.POST"
    fi
  done
}

# Run all the above tasks in parallel.
# Each task is responsible of its own checks and sampling ratio / delay.
report_df &
report_du &
report_top &
report_free &
report_iotop &
report_sensors &
report_net_dev &
report_vcgencmd &
report_diskstats &
report_nvidia_smi &
report_intel_gpu_top &
post_lines_to_influxdb &

# HACK: sleep for a while before leaving all the above tasks running in the
# background. This is so that this can be ended with Ctrl+C.
sleep 10000000000
```

## Additional scripts

### `conmon-speedtest`

```bash
#!/bin/bash

# InfluxDB target.
TARGET='http://lexicon:8086'
DBNAME=monitoring

host=$(hostname)

# Data file for batch POST.
DATA="/dev/shm/$$.txt"

tmp=/dev/shm/speedtest
speedtest-cli --secure >$tmp
netspd_up=$(grep -E 'Upload' $tmp | cut -f2 -d: | egrep -o '[0-9]+\.[0-9]')
netspd_down=$(cat ${tmp} | egrep 'Download' | cut -f2 -d: | egrep -o '[0-9]+\.[0-9]')
netspd_ping=$(cat ${tmp} | egrep 'ms$' | cut -f2 -d: | egrep -o '[0-9]+\.[0-9]')

echo "inet_up,host=${host} value=${netspd_up}" >>"$DATA"
echo "inet_down,host=${host} value=${netspd_down}" >>"$DATA"
echo "inet_ping,host=${host} value=${netspd_ping}" >>"$DATA"

# POST all data points in one batch request.
curl >/dev/null 2>/dev/null -i -XPOST "${TARGET}/write?db=${DBNAME}" --data-binary @"$DATA"
rm -f "${DATA}"
```

### `conmon-mystrom`

```bash
#!/bin/bash
#
# Export system monitoring metrics to influxdb.

# InfluxDB target.
DBNAME=monitoring
TARGET='http://lexicon:8086'

# MyStrom switches.
declare -A switches=(
    ["office"]="192.168.0.191"
)

# Default delay between each POST request to InfluxDB.
DELAY_POST=4

# Data file for batch POST.
DDIR="/dev/shm/$$"
DATA="${DDIR}/DATA.txt"
mkdir -p "${DDIR}"

host=$(hostname)

timestamp_ns() {
    date +'%s%N'
}

store_line() {
    # Write a line of data to the temporary in-memory file.
    # Exit immediately if this fails.
    echo $1 >>"${DATA}" || exit 1
}

post_lines_to_influxdb() {
    # POST data to InfluxDB in batch, when target is available.
    # Depends on: nc.
    sleep ${DELAY_POST}
    host_and_port=$(echo "${TARGET}" | sed 's/.*\///' | tr : ' ')
    if nc 2>&1 -zv ${host_and_port} | grep -q succeeded; then
        # All other tasks write data to the file in append mode (>>).
        # This task reads everything at once and immediately deletes the file.
        # This makes all the other tasks write to the same file, created anew.
        # To avoid losing data between reading and delete the file, rename it,
        # wait a little (DELAY_FAST) for the writes of tasks that already had
        # it open, then other tasks will have to write to a new file.
        mv -f "${DATA}" "${DATA}.POST"
        cut -f1 -d, "${DATA}.POST" | sort | uniq -c
        curl >/dev/null 2>/dev/null -i -XPOST "${TARGET}/write?db=${DBNAME}" --data-binary @"${DATA}.POST"
    fi
}

# myStrom WiFi switch.
# API: https://api.mystrom.ch/
# Depends on: curl.
while true; do
    for name in "${!switches[@]}"; do
        ts=$(timestamp_ns)
        switch_ip="${switches[$name]}"
        json=$(curl 2>/dev/null http://${switch_ip}/report)
        for metric in Ws power temperature; do
            value=$(echo ${json} | grep -o ".${metric}.:[^,}]*" | cut -f2 -d:)
            store_line "mystrom,switch=${name},metric=${metric} value=${value} ${ts}"
        done
    done
    post_lines_to_influxdb
done
```