# apatchをdocker run

```
docker run --name apa000ex1 httpd
```
→imageが入っていない場合はリモートからpullしようとする

```
docker ps
```
apa000ex1が立ち上がっていることを確認

```
docker stop apa000ex1
docker ps
```
apa000ex1が停止することを確認

```
docker rm apa000ex1
docker ps -a
```
apa000ex1が削除されていることを確認