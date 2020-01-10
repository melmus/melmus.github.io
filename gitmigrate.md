При наличии файла list_git, и списка репозитариев в нем, происходит клонирование репозитория, и добавления new-origin со всеми бранч-ветками и дальнейшим push в новое место гита (в примере: git@git.example.com)
```bash
#!/bin/bash
set -x
for reponame in $(cat /home/%username%/list_git)
        do
                git clone --mirror git@git.example.com:$reponame
                cd echo $reponame | rev | cut -d '/' -f 1 | rev
                git remote add new-origin ssh://git@git.example2.com/${reponame}
                git push --all new-origin
                cd ..
        done
set +x
```
Содержимое list_git

Пример файла list_git
```
testproject/service/tools/testproject.git
testproject/service/tools/util.git
testproject/service/tools/version-util.git
testproject/service/tools/tools1.git
testproject/service/tools/tool2.git
```