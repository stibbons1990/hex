---
title: Continuous Monitoring
permalink: /conmon/
---

## About Continuous Monitoring

This is the latest, most complete version of the scripts for
[Detailed system and process monitoring](blog/posts/2020-03-21-detailed-system-and-process-monitoring.md).

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
*  [`conmon-tapo.py`](#conmon-tapopy) reports temperature, humidity,
   power consumption and other monitoring data from TAPO devices.
   *  [`tapo.yaml`](#tapoyaml) is a minimal configuration file for
      `conmon-tapo.py`.

## Kubernetes Setup

This setup has been 
[migrated](blog/posts/2024-04-20-monitoring-with-influxdb-and-grafana-on-kubernetes.md)
to run on a
[single-node Kubernetes cluster](blog/posts/2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md)
and the scripts have been updated to support dual-targeting
InfluxDB over HTTP without auth and/or HTTPS with Basic Auth.

## Install InfluxDB

[Install InfluxDB OSS](https://docs.influxdata.com/influxdb/v1/introduction/install/)
or simply

```
# apt install influxdb influxdb-client -y
```

[Set it up](https://docs.influxdata.com/influxdb/v1/introduction/get-started/)
and test 
[test writing data](https://docs.influxdata.com/influxdb/v1/guides/write_data/).

Create a `monitoring` database (name can be different,
then update the `DBNAME` variable in the `conmon` scripts).

## Install Conmon

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

### The future of Flux

Flux is going into maintenance mode. You can continue
using it as you currently are without any changes to
your code.

Flux is going into maintenance mode and will not be
supported in InfluxDB 3.0. This was a decision based
on the broad demand for SQL and the continued growth
and adoption of InfluxQL. We are continuing to
support Flux for users in 1.x and 2.x so you can
continue using it with no changes to your code.
If you are interested in transitioning to InfluxDB
3.0 and want to future-proof your code, we suggest
using InfluxQL.

For information about the future of Flux, see the following:

*  [The plan for InfluxDB 3.0 Open Source](https://www.influxdata.com/blog/the-plan-for-influxdb-3-0-open-source/)
*  [InfluxDB 3.0 benchmarks](https://www.influxdata.com/blog/influxdb-3-0-is-2.5x-45x-faster-compared-to-influxdb-open-source/)

## Install Grafana

To visualize monitoring in this setup
[Grafana](https://grafana.com/grafana/)
is used.

Follow the steps to
[install Grafana on Ubuntu](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/),
[start the server with systemd](https://grafana.com/docs/grafana/latest/setup-grafana/start-restart-grafana/#start-the-grafana-server-with-systemd)
and reset its `admin` password:

```
# grafana-cli admin reset-admin-password \
  PLEASE_CHOOSE_A_SENSIBLE_PASSWORD
INFO[03-20|15:02:11] Connecting to DB                         logger=sqlstore dbtype=sqlite3
INFO[03-20|15:02:11] Starting DB migrations                   logger=migrator
Admin password changed successfully ✔
```

[Add your InfluxDB data source to Grafana](https://grafana.com/docs/grafana/latest/getting-started/get-started-grafana-influxdb/#add-your-influxdb-data-source-to-grafana),
create a new Dashboard and **Add > Visualization** for each measurement.

Finally, enable
[anonymous authentication](https://grafana.com/docs/grafana/latest/setup-grafana/start-restart-grafana/#start-the-grafana-server-with-systemd).
by tweaking `/etc/grafana/grafana.ini` as follows:

```ini
#################################### Anonymous Auth ######################
[auth.anonymous]
# enable anonymous access
enabled = true

# specify organization name that should be used for unauthenticated users
org_name = Main Org.

# specify role for unauthenticated users
org_role = Viewer

# systemctl restart grafana-server.service
```

## Install Conmon

## Single-thread

### `deploy-to-rpis`

In environments with multiple Raspberry Pi computers, it may
be useful to use this script (`deploy-to-rpis`) to deploy the
latest version of the script to all computers at once.

``` bash linenums="1" title="deploy-to-rpis"
#!/bin/bash
#
# Deplay conmon-st to Raspberry Pi hosts.

for host in pi-f1 pi-z1 pi-z2 pi3a; do
  if nc 2>&1 -zv ${host} 22 | grep -q succeeded; then
    echo "Deploying to ${host} ..."
    scp 2>/dev/null \
      -qr \
      ../conmon \
      pi@${host}:src/
    ssh 2>/dev/null \
      pi@${host} \
      "sudo cp /home/pi/src/conmon/conmon-st /usr/local/bin/conmon"
    ssh 2>/dev/null \
      pi@${host} \
      "sudo systemctl restart conmon.service"
    etc_influxdb_auth=/etc/conmon/influxdb-auth
    for src_influxdb_auth in /etc/conmon/influxdb-auth "${HOME}/.conmon-influxdb-auth"; do
      if [ -f $src_influxdb_auth ]; then
        auth=$(cat $src_influxdb_auth 2>/dev/null)
        ssh 2>/dev/null \
          pi@${host} \
          "sudo mkdir -p $(dirname $etc_influxdb_auth)"
        ssh 2>&1 >/dev/null \
          pi@${host} \
          "echo '${auth}' | sudo tee $etc_influxdb_auth"
        ssh 2>/dev/null \
          pi@${host} \
          "sudo chmod 400 $etc_influxdb_auth"
      fi
    done
  fi
done
```

### `conmon-st`

``` bash linenums="1" title="conmon-st"
#!/bin/bash
#
# Export system monitoring metrics to influxdb.

# InfluxDB target.
DBNAME=monitoring
TARGET_HTTP='' # Leave empty to skip.
TARGET_HTTPS='http://lexicon:30086'

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
  # Depends on: jq, sensors.
  jq=$(command -v jq)
  if [ -z "${jq}" ] || [ ! -f "${jq}" ]; then
    return
  fi
  sensors=$(command -v sensors)
  if [ -z "${sensors}" ] || [ ! -f "${sensors}" ]; then
    return
  fi
  sensors_json="${DDIR}/sensors"
  ts=$(timestamp_ns)
  "${sensors}" -j >"${sensors_json}"
  $jq 'keys' "${sensors_json}" | grep '^  "' | cut -f2 -d'"' | while read adapter; do
    echo "adapter: $adapter"
    $jq ".\"${adapter}\"" "${sensors_json}" | $jq 'keys' | grep '^  "' | grep -v '"Adapter"' | cut -f2 -d'"' | while read name; do
      key=$($jq ".\"${adapter}\".\"${name}\"" "${sensors_json}" | $jq 'keys' | grep '^  "' | grep '_input"' | cut -f2 -d'"')
      value=$($jq ".\"${adapter}\".\"${name}\".\"${key}\"" "${sensors_json}")
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
  # All other tasks write data to the file in append mode (>>).
  # This task reads everything at once and immediately deletes the file.
  # This makes all the other tasks write to the same file, created anew.
  sleep ${DELAY_POST}
  mv -f "${DATA}" "${DATA}.POST"
  # Post over HTTP without auth.
  if [ -n "$TARGET_HTTP" ]; then
    host_and_port=$(echo "$TARGET_HTTP" | sed 's/.*\///' | tr : ' ')
    if nc 2>&1 -zv $host_and_port | grep -q succeeded; then
      curl >/dev/null 2>/dev/null \
        -i -XPOST "${TARGET_HTTP}/write?db=${DBNAME}" \
        --data-binary @"${DATA}.POST"
    fi
  fi
  # Post over HTTPS with Basic Auth, provided credentials are
  # found in /etc/conmon/influxdb-auth
  if [ -n "$TARGET_HTTPS" ]; then
    influxdb_auth=/etc/conmon/influxdb-auth
    if [ -z $influxdb_auth ] || [ ! -f $influxdb_auth ]; then
      return
    fi
    host_and_port=$(echo "$TARGET_HTTPS" | sed 's/.*\///' | tr : ' ')
    if nc 2>&1 -zv $host_and_port | grep -q succeeded; then
      curl >/dev/null 2>/dev/null \
        -u $(sudo cat $influxdb_auth) \
        -i -XPOST "${TARGET_HTTPS}/write?db=${DBNAME}" \
        --data-binary @"${DATA}.POST"
    fi
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

## Multi-thread

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

``` bash linenums="1" title="deploy-to-pcs"
#!/bin/bash
#
# Deplay conmon-mt to PC hosts.

for host in computer smart-computer lexicon rapture; do
  if nc 2>&1 -zv ${host} 22 | grep -q succeeded; then
    echo "Deploying to ${host} ..."
    scp 2>/dev/null \
      -qr \
      ../conmon \
      root@${host}:
    ssh 2>/dev/null \
      root@${host} \
      "cp /root/conmon/conmon-mt /usr/local/bin/conmon"
    ssh 2>/dev/null \
      root@${host} \
      "systemctl restart conmon.service"
    etc_influxdb_auth=/etc/conmon/influxdb-auth
    for src_influxdb_auth in /etc/conmon/influxdb-auth "${HOME}/.conmon-influxdb-auth"; do
      if [ -f $src_influxdb_auth ]; then
        ssh 2>/dev/null \
          root@${host} \
          "mkdir -p $(dirname $etc_influxdb_auth)"
        scp 2>/dev/null \
          -qr \
          $src_influxdb_auth \
          root@${host}:$etc_influxdb_auth
        ssh 2>/dev/null \
          root@${host} \
          "chmod 400 $etc_influxdb_auth"
      fi
    done
  fi
done
```

### `conmon-mt`

``` bash linenums="1" title="conmon-mt"
#!/bin/bash
#
# Export system monitoring metrics to influxdb.

# InfluxDB target.
DBNAME=monitoring
TARGET_HTTP='' # Leave empty to skip.
TARGET_HTTPS='http://lexicon:30086'

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
  # Depends on: jq, sensors.
  jq=$(command -v jq)
  if [ -z "${jq}" ] || [ ! -f "${jq}" ]; then
    return
  fi
  sensors=$(command -v sensors)
  if [ -z "${sensors}" ] || [ ! -f "${sensors}" ]; then
    return
  fi
  sensors_json="${DDIR}/sensors"
  while true; do
    ts=$(timestamp_ns)
    "${sensors}" -j >"${sensors_json}"
    $jq 'keys' "${sensors_json}" | grep '^  "' | cut -f2 -d'"' | while read adapter; do
      echo "adapter: $adapter"
      $jq ".\"${adapter}\"" "${sensors_json}" | $jq 'keys' | grep '^  "' | grep -v '"Adapter"' | cut -f2 -d'"' | while read name; do
        key=$($jq ".\"${adapter}\".\"${name}\"" "${sensors_json}" | $jq 'keys' | grep '^  "' | grep '_input"' | cut -f2 -d'"')
        value=$($jq ".\"${adapter}\".\"${name}\".\"${key}\"" "${sensors_json}")
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
  # POST data to InfluxDB in batch, when targets are available.
  # Depends on: curl, nc.
  # All other tasks write data to the file in append mode (>>).
  # This task reads everything at once and immediately deletes the file.
  # This makes all the other tasks write to the same file, created anew.
  # To avoid losing data between reading and delete the file, rename it,
  # wait a little (DELAY_FAST) for the writes of tasks that already had
  # it open, then other tasks will have to write to a new file.
  while true; do
    sleep ${DELAY_POST}
    mv -f "${DATA}" "${DATA}.POST"
    # Post over HTTP without auth.
    if [ -n "$TARGET_HTTP" ]; then
      host_and_port=$(echo "$TARGET_HTTP" | sed 's/.*\///' | tr : ' ')
      if nc 2>&1 -zv $host_and_port | grep -q succeeded; then
        curl >/dev/null 2>/dev/null \
          -i -XPOST "${TARGET_HTTP}/write?db=${DBNAME}" \
          --data-binary @"${DATA}.POST"
      fi
    fi
    # Post over HTTPS with Basic Auth, provided credentials are
    # found in /etc/conmon/influxdb-auth
    if [ -n "$TARGET_HTTPS" ]; then
      influxdb_auth=/etc/conmon/influxdb-auth
      if [ -z $influxdb_auth ] || [ ! -f $influxdb_auth ]; then
        return
      fi
      host_and_port=$(echo "$TARGET_HTTPS" | sed 's/.*\///' | tr : ' ')
      if nc 2>&1 -zv $host_and_port | grep -q succeeded; then
        curl >/dev/null 2>/dev/null \
          -u $(sudo cat $influxdb_auth) \
          -i -XPOST "${TARGET_HTTPS}/write?db=${DBNAME}" \
          --data-binary @"${DATA}.POST"
      fi
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

``` bash linenums="1" title="conmon-speedtest"
#!/bin/bash

# InfluxDB target.
DBNAME=monitoring
TARGET_HTTP='' # Leave empty to skip.
TARGET_HTTPS='http://lexicon:30086'

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
# Post over HTTP without auth.
if [ -n "$TARGET_HTTP" ]; then
    host_and_port=$(echo $TARGET_HTTP | sed 's/.*\///' | tr : ' ')
    if nc 2>&1 -zv $host_and_port | grep -q succeeded; then
        curl >/dev/null 2>/dev/null \
            -i -XPOST "${TARGET_HTTP}/write?db=${DBNAME}" \
            --data-binary @"${DATA}"
    fi
fi
# Post over HTTPS with Basic Auth, provided credentials are
# found in /etc/conmon/influxdb-auth
if [ -n "$TARGET_HTTPS" ]; then
    influxdb_auth=/etc/conmon/influxdb-auth
    if [ -z $influxdb_auth ] || [ ! -f $influxdb_auth ]; then
        exit 1
    fi
    host_and_port=$(echo $TARGET_HTTPS | sed 's/.*\///' | tr : ' ')
    if nc 2>&1 -zv $host_and_port | grep -q succeeded; then
        curl >/dev/null 2>/dev/null \
            -u $(cat $influxdb_auth) \
            -i -XPOST "${TARGET_HTTPS}/write?db=${DBNAME}" \
            --data-binary @"${DATA}"
    fi
fi
rm -f "${DATA}"
```

### `conmon-mystrom`

``` bash linenums="1" title="conmon-mystrom"
#!/bin/bash
#
# Export system monitoring metrics to influxdb.

# InfluxDB target.
DBNAME=monitoring
TARGET_HTTP='' # Leave empty to skip.
TARGET_HTTPS='http://lexicon:30086'

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
    # All other tasks write data to the file in append mode (>>).
    # This task reads everything at once and immediately deletes the file.
    # This makes all the other tasks write to the same file, created anew.
    # To avoid losing data between reading and delete the file, rename it,
    # wait a little (DELAY_FAST) for the writes of tasks that already had
    # it open, then other tasks will have to write to a new file.
    # Post over HTTP without auth.
    sleep ${DELAY_POST}
    if [ -n "$TARGET_HTTP" ]; then
        host_and_port=$(echo $TARGET_HTTP | sed 's/.*\///' | tr : ' ')
        if nc 2>&1 -zv $host_and_port | grep -q succeeded; then
            curl >/dev/null 2>/dev/null \
                -i -XPOST "${TARGET_HTTP}/write?db=${DBNAME}" \
                --data-binary @"${DATA}"
        fi
    fi
    # Post over HTTPS with Basic Auth, provided credentials are
    # found in /etc/conmon/influxdb-auth
    if [ -n "$TARGET_HTTPS" ]; then
        influxdb_auth=/etc/conmon/influxdb-auth
        if [ -z $influxdb_auth ] || [ ! -f $influxdb_auth ]; then
            return
        fi
        host_and_port=$(echo $TARGET_HTTPS | sed 's/.*\///' | tr : ' ')
        if nc 2>&1 -zv $host_and_port | grep -q succeeded; then
            curl >/dev/null 2>/dev/null \
                -u $(cat $influxdb_auth) \
                -i -XPOST "${TARGET_HTTPS}/write?db=${DBNAME}" \
                --data-binary @"${DATA}"
        fi
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

### `conmon-tapo.py`

[Continuous Monitoring for TP-Link Tapo devices](blog/posts/2024-12-28-continuous-monitoring-for-tp-link-tapo-devices.md)
explains this script and its dependencies in more details.

``` py linenums="1"
#!/usr/bin/env python3

"""Script to poll TAPO devices for monitoring data and post it to InfluxDB.

Run conmon-tapo --help to see all available options.

Usage:
  conmon-tapo [options]

Requires:
  absl-py
  https://github.com/abseil/abseil-py

  tapo
  https://github.com/mihai-dinculescu/tapo/
"""

import asyncio
import json
import os
import socket
import sys
import time
import yaml

from absl import app, flags
from datetime import datetime
from influxdb import InfluxDBClient

from tapo import ApiClient
from tapo.requests import EnergyDataInterval
from tapo.responses import T31XResult

FLAGS = flags.FLAGS

flags.DEFINE_string(
    "config",
    "/etc/conmon/tapo.yaml",
    "Configuration file with settings for InfluxDB and Tapo devices.",
    short_name="c"
)


def load_config(filepath):
    with open(filepath) as f:
        config = yaml.load(f, Loader=yaml.FullLoader)
    # Override values from environment variables.
    for section in ("influxdb", "tapo_auth"):
        for variable in config[section]:
            if variable[-8:] in ("username", "password"):
                env_value = os.getenv(config[section][variable])
                if env_value:
                    config[section][variable] = env_value
    return config


async def fetch_reports(config):
    client = ApiClient(**config["tapo_auth"])
    reports = []
    for device in config["devices"]:
        model = device["model"]
        report=dict(model=model)
        try:
            if model == "H100":
                device_conn = await client.h100(device["ip"])
                device_info = await device_conn.get_device_info()
                report=dict(
                    model=device_info.to_dict().get("model"),
                    nickname=device_info.to_dict().get("nickname")
                )
                children=[]
                child_device_list = await device_conn.get_child_device_list()
                for child in child_device_list:
                    if isinstance(child, T31XResult):
                        t315 = await device_conn.t315(device_id=child.device_id)
                        children.append(dict(
                            model="T315",
                            nickname=child.nickname,
                            humidity=child.current_humidity,
                            temperature=child.current_temperature
                        ))
                report.update(dict(children=children))
            elif model in ("P110", "P115"):
                device_conn = await client.p110(device["ip"])
                device_info = await device_conn.get_device_info()
                energy_usage = await device_conn.get_energy_usage()
                report=dict(
                    model=device_info.to_dict().get("model"),
                    nickname=device_info.to_dict().get("nickname"),
                    current_power=float(energy_usage.to_dict().get("current_power")/1000)
                )
        except:
            print("Could not get data from %s on %s " % (
                device["model"], device["ip"]))
        else:
            reports.append(report)
    return reports


def always_on_reports(config):
    reports = []
    if "always_on" not in config:
      return reports
    for device in config["always_on"]:
        reports.append(dict(
            model="P115",
            nickname=device["name"],
            current_power=float(device["power"])
        ))
    return reports


def json_body_point(measurement, value, ts, tags):
    tags.update(dict(host=socket.gethostname()))
    return {
      "measurement": measurement,
      "tags": tags,
      "time": ts,
      "fields": {
        "value": value
      }
    }


def json_body_points_from_report(report, ts):
    json_body = []
    tags = dict(model=report["model"], nickname=report["nickname"])
    for field in report:
        if field in ("model", "nickname"): continue
        json_body.append(json_body_point("tapo_%s" % field, report[field], ts, tags))
    return json_body


def post_reports(config, reports):
    client = InfluxDBClient(**config["influxdb"])
    json_body = []
    ts = int(1000000000 * time.mktime(time.localtime()))
    for report in reports:
        if "children" in report:
            for child in report["children"]:
                json_body.extend(json_body_points_from_report(child, ts))
        else:
            json_body.extend(json_body_points_from_report(report, ts))
    client.write_points(json_body)


def main(argv):
    config = load_config(FLAGS.config)
    reports = asyncio.run(fetch_reports(config))
    reports.extend(always_on_reports(config))
    post_reports(config, reports)


if __name__ == "__main__":
    app.run(main)
```

#### `tapo.yaml`

``` yaml linenums="1"
always_on:
  - name: "Dehumidifier"
    power: "250"
  - name: "PC"
    power: "150"
devices:
  - ip: "192.168.0.115"
    model: "P115"
  - ip: "192.168.0.100"
    model: "H100"
influxdb:
  host: inf.ssl.uu.am
  port: 443
  database: "home"
  username: "INFLUXDB_USERNAME"
  password: "INFLUXDB_PASSWORD"
  ssl: True
  verify_ssl: True
tapo_auth:
  tapo_username: "TAPO_USERNAME"
  tapo_password: "TAPO_PASSWORD"
```

#### Python dependencies

The `absl` and `influxdb` modules can be installed with`apt`:

``` console
# apt install python3-absl python3-influxdb
```

The [`tapo`](https://github.com/mihai-dinculescu/tapo/) module
is more complicated to install. The recommended approach is to use
**virtualenv**:

??? terminal "`# apt install python3-venv -y`"

    ``` console
    # apt install python3-venv -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following additional packages will be installed:
      python3-pip-whl python3-setuptools-whl python3.12-venv
    The following NEW packages will be installed:
      python3-pip-whl python3-setuptools-whl python3-venv python3.12-venv
    0 upgraded, 4 newly installed, 0 to remove and 3 not upgraded.
    Need to get 2,425 kB of archives.
    After this operation, 2,777 kB of additional disk space will be used.
    Get:1 http://archive.ubuntu.com/ubuntu noble-updates/universe amd64 python3-pip-whl all 24.0+dfsg-1ubuntu1.1 [1,703 kB]
    Get:2 http://archive.ubuntu.com/ubuntu noble-updates/universe amd64 python3-setuptools-whl all 68.1.2-2ubuntu1.1 [716 kB]
    Get:3 http://archive.ubuntu.com/ubuntu noble-updates/universe amd64 python3.12-venv amd64 3.12.3-1ubuntu0.3 [5,678 B]
    Get:4 http://archive.ubuntu.com/ubuntu noble-updates/universe amd64 python3-venv amd64 3.12.3-0ubuntu2 [1,034 B]
    Fetched 2,425 kB in 1s (1,736 kB/s)      
    Selecting previously unselected package python3-pip-whl.
    (Reading database ... 425834 files and directories currently installed.)
    Preparing to unpack .../python3-pip-whl_24.0+dfsg-1ubuntu1.1_all.deb ...
    Unpacking python3-pip-whl (24.0+dfsg-1ubuntu1.1) ...
    Selecting previously unselected package python3-setuptools-whl.
    Preparing to unpack .../python3-setuptools-whl_68.1.2-2ubuntu1.1_all.deb ...
    Unpacking python3-setuptools-whl (68.1.2-2ubuntu1.1) ...
    Selecting previously unselected package python3.12-venv.
    Preparing to unpack .../python3.12-venv_3.12.3-1ubuntu0.3_amd64.deb ...
    Unpacking python3.12-venv (3.12.3-1ubuntu0.3) ...
    Selecting previously unselected package python3-venv.
    Preparing to unpack .../python3-venv_3.12.3-0ubuntu2_amd64.deb ...
    Unpacking python3-venv (3.12.3-0ubuntu2) ...
    Setting up python3-setuptools-whl (68.1.2-2ubuntu1.1) ...
    Setting up python3-pip-whl (24.0+dfsg-1ubuntu1.1) ...
    Setting up python3.12-venv (3.12.3-1ubuntu0.3) ...
    Setting up python3-venv (3.12.3-0ubuntu2) ...
    ```

``` console
$ mkdir -p ~/.venvs
$ python3 -m venv --system-site-packages ~/.venvs/tapo
$ ~/.venvs/tapo/bin/python -m pip install tapo
Collecting tapo
  Using cached tapo-0.8.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl.metadata (12 kB)
Using cached tapo-0.8.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (3.3 MB)
Installing collected packages: tapo
Successfully installed tapo-0.8.0

$ ~/.venvs/tapo/bin/python ./conmon-tapo.py
```

The `tapo` module can also be installed the *not recommended* way: 

``` console
# pip install --break-system-packages tapo
Collecting tapo
  Using cached tapo-0.8.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl.metadata (12 kB)
Using cached tapo-0.8.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (3.3 MB)
Installing collected packages: tapo
Successfully installed tapo-0.8.0
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv
```

#### Device IP dependencies

The "`conmon-tapo.py` script was initially writen on the assumption
that Tapo devices would keep their IP address constant long-term,
so if they change IP addresses the script crashes when trying to read
the wrong fields, or simply not being able to connect to one of them:

??? failure "Trying to reach a device at nobody's IP address"

    ``` console hl_lines="25"
    $ ~/.venvs/tapo/bin/python ./conmon-tapo.py 
    Traceback (most recent call last):
      File "/home/coder/src/conmon/./conmon-tapo.py", line 138, in <module>
        app.run(main)
      File "/usr/lib/python3/dist-packages/absl/app.py", line 308, in run
        _run_main(main, args)
      File "/usr/lib/python3/dist-packages/absl/app.py", line 254, in _run_main
        sys.exit(main(argv))
                ^^^^^^^^^^
      File "/home/coder/src/conmon/./conmon-tapo.py", line 132, in main
        reports = asyncio.run(fetch_reports(config))
                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      File "/usr/lib/python3.12/asyncio/runners.py", line 194, in run
        return runner.run(main)
              ^^^^^^^^^^^^^^^^
      File "/usr/lib/python3.12/asyncio/runners.py", line 118, in run
        return self._loop.run_until_complete(task)
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      File "/usr/lib/python3.12/asyncio/base_events.py", line 687, in run_until_complete
        return future.result()
              ^^^^^^^^^^^^^^^
      File "/home/coder/src/conmon/./conmon-tapo.py", line 83, in fetch_reports
        device_conn = await client.p110(device["ip"])
                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    Exception: Http(reqwest::Error { kind: Request, url: "http://192.168.0.41/app", source: hyper_util::client::legacy::Error(Connect, ConnectError("tcp connect error", Os { code: 113, kind: HostUnreachable, message: "No route to host" })) })
    ```

??? failure "Trying to reach a device at somebody else's IP address"

    ``` console hl_lines="25"
    $ ~/.venvs/tapo/bin/python ./conmon-tapo.py 
    Traceback (most recent call last):
      File "/home/coder/src/conmon/./conmon-tapo.py", line 138, in <module>
        app.run(main)
      File "/usr/lib/python3/dist-packages/absl/app.py", line 308, in run
        _run_main(main, args)
      File "/usr/lib/python3/dist-packages/absl/app.py", line 254, in _run_main
        sys.exit(main(argv))
                ^^^^^^^^^^
      File "/home/coder/src/conmon/./conmon-tapo.py", line 132, in main
        reports = asyncio.run(fetch_reports(config))
                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      File "/usr/lib/python3.12/asyncio/runners.py", line 194, in run
        return runner.run(main)
              ^^^^^^^^^^^^^^^^
      File "/usr/lib/python3.12/asyncio/runners.py", line 118, in run
        return self._loop.run_until_complete(task)
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      File "/usr/lib/python3.12/asyncio/base_events.py", line 687, in run_until_complete
        return future.result()
              ^^^^^^^^^^^^^^^
      File "/home/coder/src/conmon/./conmon-tapo.py", line 83, in fetch_reports
        device_conn = await client.p110(device["ip"])
                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    Exception: Http(reqwest::Error { kind: Request, url: "http://192.168.0.32/app", source: hyper_util::client::legacy::Error(SendRequest, hyper::Error(Io, Os { code: 104, kind: ConnectionReset, message: "Connection reset by peer" })) })
    ```

??? failure "Trying to read a P115 like it was an H100"

    ``` console hl_lines="25"
    $ ~/.venvs/tapo/bin/python ./conmon-tapo.py
    Traceback (most recent call last):
      File "/home/coder/src/conmon/./conmon-tapo.py", line 138, in <module>
        app.run(main)
      File "/usr/lib/python3/dist-packages/absl/app.py", line 308, in run
        _run_main(main, args)
      File "/usr/lib/python3/dist-packages/absl/app.py", line 254, in _run_main
        sys.exit(main(argv))
                ^^^^^^^^^^
      File "/home/coder/src/conmon/./conmon-tapo.py", line 132, in main
        reports = asyncio.run(fetch_reports(config))
                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      File "/usr/lib/python3.12/asyncio/runners.py", line 194, in run
        return runner.run(main)
              ^^^^^^^^^^^^^^^^
      File "/usr/lib/python3.12/asyncio/runners.py", line 118, in run
        return self._loop.run_until_complete(task)
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      File "/usr/lib/python3.12/asyncio/base_events.py", line 687, in run_until_complete
        return future.result()
              ^^^^^^^^^^^^^^^
      File "/home/coder/src/conmon/./conmon-tapo.py", line 65, in fetch_reports
        device_info = await device_conn.get_device_info()
                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    Exception: Serde(Error("missing field `in_alarm_source`", line: 1, column: 822))
    ```
