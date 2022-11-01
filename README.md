cmd

    ps -ef | grep hexo | grep -v grep | awk '{print $2}' | xargs kill
    nohup hexo server >/dev/null &
