```bash
cp -r ~/panel-backup/panel ~/.config/xfce4/
cp ~/panel-backup/xfce4-panel.xml ~/.config/xfce4/xfconf/xfce-perchannel-xml/
xfce4-panel --restart
xfconf-query -c xfce4-panel -p /panels
xfconf-query -c xfce4-panel -p /panels/panel-0/output-name
xrandr | grep connected
xfconf-query -c xfce4-panel -p /panels/panel-0/output-name -s HDMI-1

```
