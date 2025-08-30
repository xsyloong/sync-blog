> [!INFO] 网络分析工具
> WireShark 网络分析工具  分析网络网络内容

1. 在 linux 下载 `sudo apt-get install wireshark`
2. 下载过程中需要确认权限：
	1. ![[Pasted image 20250830165351.png]] 
	2. 需要点击 Yes
	3. 如果上述没有点击 Yes ，可以输入以下命令 ` sudo dpkg-reconfigure wireshark-common ` 后出现以上界面，点击 Yes
3. 输入 ` sudo vim /etc/group` 命令，翻倒最下面找到  ![[Pasted image 20250830165633.png]] 这种内容，需要在后面加入用户操作权限
	1. 此时输入 `sudo usermod -aG wireshark $USER` ，然后重启系统即可。
4. 具体使用在下文中有提及：
	1. https://www.cnblogs.com/linyfeng/p/9496126.html