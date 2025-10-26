




    ============================================================================================================
**Sysstat & systemd timers**
🔹 Purpose 
    → Sysstat = tools for collecting system performance statistics.
    → A systemd timer runs every 10 minutes to collect the data.
    --------------------------------------------------------------
    **yum install sysstat**                      → Install
    **systemctl status sysstat**                 → Check activity
        Shows active (exited) → means the service runs only when triggered by its timer
    **systemctl list-units --type=timer**        → Timer
        sysstat-collect.timer This timer triggers:  sysstat-collect.service
    **systemctl cat sysstat-collect.timer**      → Verify schedule
        Default setting: *:00/10 → every 10 minutes
        Timer unit file is located at: /usr/lib/systemd/system/sysstat-collect.timer
    <Customizing the interval>
    **cp /usr/lib/systemd/system/sysstat-collect.timer /etc/systemd/system/sysstat-collect.timer**
    **vim /etc/systemd/system/sysstat-collect.timer**
    Example change: *:00/01 → every 1 minute
    Reload systemd: **systemctl daemon-reload**
    Using SAR:      **sar -r**
    Displays memory utilization statistics collected (default: every 10 min).
    If you changed to 1 min, you’ll see data points every minute.