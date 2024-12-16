通过 colab 拉取镜像打包通过 webdav 上传到坚果云，国内主机通过 curl 下载

坚果云每月 1G 上传，理论无限空间

每月 3G 下载， 正适合轻量使用

## 1. 修改 hosts

经过我的尝试，坚果云似乎把 colab 拉黑了， 需要修改 hosts 文件中 webdav 的网站对应 ip

- 可以通过 itdog 等工具，`dav.jianguoyun.com:443` 找一个国内的 ip （速度慢）

- 也可以国外小鸡配置自己搭一个 realm 转发到 dav.jianguoyun.com:443 （速度快）

### 修改

1. 先查看自己的hosts文件(colab 是python环境，执行终端命令需要加!，左边播放按钮执行代码，或者 Shift + Enter)
	```
	!cat /etc/hosts
	```
2. 使用 echo 命令将内容追加到 /etc/hosts 文件中, 对照着，将8.8.8.8换掉
	```
	!echo -e "127.0.0.1\tlocalhost\n::1\tlocalhost ip6-localhost ip6-loopback\nfe00::0\tip6-localnet\nff00::0\tip6-mcastprefix\nff02::1\tip6-allnodes\nff02::2\tip6-allrouters\n172.28.0.12\t9445d586e4d8\n8.8.8.8\tdav.jianguoyun.com" | sudo tee /etc/hosts
	```

3. 测试能否正常访问坚果云（坚果云根目录建一个docker文件夹）
	```
	!touch 1.txt
	!curl -u 坚果云邮箱:坚果云密码 -T ./1.txt https://dav.jianguoyun.com/dav/docker/1.txt
	```
	登上坚果云看看有没有1.txt这个文件
	
## 2. 将 google drive 挂载到 colab

左边导航栏，有提示，可以将 google drive 挂载进来

根目录建一个 docker 文件夹

我是为了，同时保存到 google drive 和坚果云

不需要这个功能的，自己修改下面相关的代码

## 3. 代码
```
!pip install webdav4

def udocker_init():
    import os
    if not os.path.exists("/home/user"):
        !pip install udocker > /dev/null
        !udocker --allow-root install > /dev/null
        !useradd -m user > /dev/null
    print(f'Docker-in-Colab 1.1.0\n')
    print(f'Usage:     udocker("--help")')
    print(f'Examples:  https://github.com/indigo-dc/udocker?tab=readme-ov-file#examples')

    def execute(command: str):
        user_prompt = "\033[1;32muser@pc\033[0m"
        print(f"{user_prompt}$ udocker {command}")
        !su - user -c "udocker $command"

    return execute

udocker = udocker_init()

def save_image(platform, image_name):
    udocker("rmi " + image_name)
    udocker("pull " + "--platform=" + platform + " " + image_name)
    file_name = image_name.replace(":", "_")+"_" + platform
    file_name = file_name.replace("/", "_")+'.tar'
    udocker("save -o "+ file_name+" " + image_name)
    !gzip -c /home/user/{file_name} > /content/drive/MyDrive/docker/{file_name}.gz
    if os.path.exists(f'/content/drive/MyDrive/docker/{file_name}.gz'):
        print(f"Info: {file_name}_下载成功！")
    else:
        print(f"Info: {file_name}_下载失败！")
    udocker("rmi " + image_name)
    return file_name

def upload_to_webdav(file_path):
	
	# 如果搭建了 realm 转发，注意修改 443 为你服务器对应的端口
    webdav_url = "https://dav.jianguoyun.com:443/dav/docker/"
    webdav_username = "你的邮箱"
    webdav_password = "你的密码"

    if webdav_url and webdav_username and webdav_password:
        from webdav4.client import Client
        client = Client(webdav_url, auth=(webdav_username, webdav_password), timeout=300)
        try:
            client.upload_file(file_path, file_path.split('/')[-1])
            print(f"Info: 文件 {file_path} 上传成功！")
        except Exception as e:
            print(f"Error: 文件上传失败，错误信息：{e}")
    else:
        print("Error: WebDAV 配置信息不完整，请检查配置文件！")

# 下载一个 Docker 镜像，保存到 google drive，返回文件名
# 第一个参数指定架构（ linux/amd64 或者 linux/arm64）第二个指定标签
file_name = save_image("linux/amd64", "filebrowser/filebrowser")

# 上传文件到 WebDAV
upload_to_webdav(f'/content/drive/MyDrive/docker/{file_name}.gz')
```
	
## 4. 加载镜像

```
# webdav 下载
curl -u username:password -o filebrowser_filebrowser_linux_amd64.tar.gz https://dav.jianguoyun.com/dav/docker/filebrowser_filebrowser_linux_amd64.tar.gz
# 解压
gzip -d filebrowser_filebrowser_linux_amd64.tar.gz
# 加载镜像
docker load -i filebrowser_filebrowser_linux_amd64.tar
# 运行容器
docker run -d --name fb filebrowser/filebrowser
```

