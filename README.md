cmd
    export PATH="/home/ubuntu/download/node-v12.22.12-linux-x64/bin:$PATH"

    hexo new 202302
    rm ~/blog.md -i
    ln -s ~/src/hexo/source/_posts/202302.md  ~/blog.md

    ps -ef | grep hexo | grep -v grep | awk '{print $2}' | xargs kill
    nohup hexo server >/dev/null &
