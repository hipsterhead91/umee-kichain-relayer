# Установка релеера между KiChain и Umee

## Подготовка

Для начала открываем /root/kichain/kid/config/config.toml и убеждаемся, что laddr = "tcp://127.0.0.1:26657" (исправляем, если это не так). Если пришлось внести изменения, то перезапускаем сервис:

>sudo systemctl restart kichaind

Проверяем, что всё в порядке: 

>kid status 2>&1 | jq

Если ранее на сервере был установлен релеер, удаляем папку с ним (/root/.relayer).

## Установка и настройка релеера

Качаем и распаковываем архив:

>wget https://github.com/cosmos/relayer/releases/download/v0.9.3/Cosmos.Relayer_0.9.3_linux_amd64.tar.gz

>tar -zxvf Cosmos.Relayer_0.9.3_linux_amd64.tar.gz

>cp "Cosmos Relayer" /usr/local/bin/rly

Инициализируем релеер:

>cp "Cosmos Relayer" /usr/local/bin/rly

Вручную редактируем config.yaml:

    global:
      api-listen-addr: :5183
      timeout: 3m
      light-cache-size: 20
    chains:
    - key:
      chain-id: umee-betanet-1
      rpc-addr: http://161.97.78.75:26657
      account-prefix: umee
      gas-adjustment: 1.5
      gas-prices: 0.025uumee
      trusting-period: 10m
    - key:
      chain-id: kichain-t-4
      rpc-addr: http://127.0.0.1:26657
      account-prefix: tki
      gas-adjustment: 1.5
      gas-prices: 0.025utki
      trusting-period: 10m
    paths: {}

Создаём ключи для Umee:

>rly keys add umee-betanet-1 umeekey

Вывод будет примерно таким:

    {"mnemonic":"xxx xxx xxx xxx xxx ...":"umee1ujnm9g8xtvlhjjxr0twnv227ma4cjkXXXXXXX"}

Восстанавливаем ключи от кичейна:

>rly keys restore kichain-t-4 kikey "<put your mnemonic here divided by space ....>" #restores

Вывод:

    {"mnemonic":"xxx xxxx xxx xxx xx ...":"tki1pwj3kxmkaz45y0gd9zxyaprd4cvx3XXXXXXXXX"}

Добавляем ключи в чейн:

>rly chains edit umee-betanet-1 key umeekey
  
>rly chains edit kichain-t-4 key kikey

Далее нужно "подоить" кран, ссылку на который можно найти в дискорде Umee, после чего проверяем баланс:

>rly query balance umee-betanet-1
  
>rly query balance kichain-t-4

Вывод: 

    root@vmi647657:~# rly query balance umee-betanet-1
    100000000uumee
    root@vmi647657:~# rly query balance kichain-t-4
    90755269utki

## Запуск

Запускаем клиенты обеих сетей:

>rly light init umee-betanet-1 -f
  
>rly light init kichain-t-4 -f

Вывод соответственно: 

    root@vmi657940:~# rly light init umee-betanet-1 -f
    successfully created light client for umee-betanet-1 by trusting endpoint http://161.97.78.75:26657
    root@vmi657940:~# rly light init kichain-t-4 -f
    successfully created light client for kichain-t-4 by trusting endpoint http://127.0.0.1:26657...

Создаем путь UMEE to KI:

>rly paths generate umee-betanet-1 kichain-t-4 umee_to_ki_path --port=transfer

Вывод:

    Generated path(umee_to_ki_path), run 'rly paths show umee_to_ki_path --yaml' to see details

Проверяем, что всё в порядке (везде должны быть галочки):
  
>rly chains list

Вывод:

    0: umee-betanet-1       -> key(✔) bal(✔) light(✔) path(✔)
    1: kichain-t-4          -> key(✔) bal(✔) light(✔) path(✔)

Открываем канал UMEE to KI:

>rly tx link umee_to_ki_path

Ждём, когда в терминале появится "Channel created". Если канал не открылся, повторяем команду. Вывод должен быть таким:

    I[2021-09-09|12:15:56.731] ★ Clients created: client(07-tendermint-1) on chain[umee-betanet-1] and client(07-tendermint-13) on chain[kichain-t-4]
    I[2021-09-09|12:15:57.129] ★ Connection created: [umee-betanet-1]client{07-tendermint-1}conn{connection-1} -> [kichain-t-4]client{07-tendermint-13}conn{connection-17}
    I[2021-09-09|12:15:57.437] ★ Channel created: [umee-betanet-1]chan{channel-0}port{transfer} -> [kichain-t-4]chan{channel-61}port{transfer}

Отправляем токены с UMEE на KI:

>rly tx transfer umee-betanet-1 kichain-t-4 1000000uumee tki1pwj3kxmkaz45y0gd9zxyaprd4cvxXXXXXXXX --path umee_to_ki_path

Вывод:

    I[2021-09-09|14:06:42.133] ✔ [umee-betanet-1]@{239955} - msg(0:transfer) hash(2990469E35B078952DC1A524BC0DB43CD46DDF6F5829015BE36DD902D33EF364)

Проверяем баланс:

>rly query balance kichain-t-4

Вывод:

    root@vmi657940:~# rly query balance kichain-t-4
    10000000utki

Если баланс не обновился, открываем config.yaml и правим две строчки "channel-id", чтобы получилось так:

    paths:
      umee_to_ki_path:
        src:
          chain-id: umee-betanet-1
          client-id: 07-tendermint-9
          connection-id: connection-54
          channel-id: channel-0
          port-id: transfer
          order: UNORDERED
          version: ics20-1
        dst:
          chain-id: kichain-t-4
          client-id: 07-tendermint-290
          connection-id: connection-303
          channel-id: channel-61
          port-id: transfer
          order: UNORDERED
          version: ics20-1
        strategy:
          type: naive

Снова открываем канал и ждём "Channel created", повторяем если не получилось:

>rly tx link umee_to_ki_path

И снова осуществляем транзакцию:

>rly tx transfer umee-betanet-1 kichain-t-4 1000000uumee tki1pwj3kxmkaz45y0gd9zxyaprd4cvXXXXXXXXX --path umee_to_ki_path

Теперь баланс должен измениться (может занять около 10 секунд):

>rly query balance kichain-t-4

Далее будем отправляеть транзакции с KI на UMEE. Сперва обновляем клиенты:

>rly light init umee-betanet-1 -f
  
>rly light init kichain-t-4 -f

Создаём обратный путь KI to UMEE:

>rly paths generate kichain-t-4 umee-betanet-1 ki_to_umee_path --port=transfer

Открываем канал KI to UMEE, ждём "Channel created", повторяем если не создаст канал:

>rly tx link ki_to_umee_path

Проверяем баланс:

>rly query balance umee-betanet-1

Выполняем транзакцию:

>rly tx transfer kichain-t-4 umee-betanet-1 1000000utki umee1ujnm9g8xtvlhjjxr0twnv227ma4XXXXXXXXXX --path ki_to_umee_path

Если баланс не обновился, открываем config.yaml и правим две строчки "channel-id", чтобы получилось так:

    paths:
      ki_to_umee_path:
        src:
          chain-id: kichain-t-4
          client-id: 07-tendermint-295
          connection-id: connection-308
          channel-id: channel-61
          port-id: transfer
          order: UNORDERED
          version: ics20-1
        dst:
          chain-id: umee-betanet-1
          client-id: 07-tendermint-9
          connection-id: connection-60
          channel-id: channel-0
          port-id: transfer
          order: UNORDERED
          version: ics20-1
        strategy:
          type: naive

Снова открываем канал KI to UMEE, снова ждём "Channel created", повторяем если не создаст канал:

>rly tx link ki_to_umee_path

Снова транзакция:

>rly tx transfer kichain-t-4 umee-betanet-1 1000000utki umee1ujnm9g8xtvlhjjxr0twnv227ma4XXXXXXXX --path ki_to_umee_path

Снова проверяем баланс:

>rly query balance umee-betanet-1

Повторяем транзакции 5 раз, хэши куда-нибудь сохраняем. Всё готово!

## Возможные проблемы

Если после "rly tx transfer" возникает подобная ошибка:

    Error: failed to get trusted header, please ensure header at the height 299076 has not been pruned by the connected node: verify from #298754 to #299077 failed: old header has expired at 2021-09-12 14:15:58.146755392 +0000 UTC (now: 2021-09-12 16:48:53.926184772 +0200 CEST m=+0.296514904)

то мы удаляем папку light в директории /root/.relayer и выполняем следующие команды:

>rly light init umee-betanet-1 -f
  
>rly light init kichain-t-4 -f
  
>rly tx link <путь>

Если после "rly tx link <путь>" возникает подобная ошибка:

    Error: failed to update off-chain light client for chain kichain-t-4: verify from #298003 to #298106 failed: old header has expired at 2021-09-12 12:54:45.10796825 +0000 UTC (now: 2021-09-12 14:55:48.593232859 +0200 CEST m=+0.323576592)

То мы обновляем клиенты и снова повторяем команду:

>rly light init umee-betanet-1 -f
  
>rly light init kichain-t-4 -f
  
>rly tx link <путь>
