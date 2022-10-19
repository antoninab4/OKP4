

# OKP4 Ручная настройка узла



Оригинальный документ:
>- [Оригинальный документ](https://docs.okp4.network/docs/nodes/installation)

Explorer:
>- https://explorer.bccnodes.com/okp4


## Установите необходимые обновления и инструменты
```
sudo apt update && sudo apt upgrade -y
```
```
sudo apt install curl build-essential git wget jq make gcc tmux chrony -y
```
## Установить Go (одна команда)
```
if ! [ -x "$(command -v go)" ]; then
  ver="1.18.2"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi
```
## Присвоим нашей ноде имя (скобки убираем)
```
NODENAME=<имя_ноды>
```

## Сделайте копию репозитория github и установите
```
cd $HOME
git clone https://github.com/okp4/okp4d.git
cd okp4d
make install
```

## Проверим версию; Должно быть v2.2.0
```
okp4d version
```

## Давайте подготовимся к запуску Node
```
okp4d config chain-id okp4-nemeton
okp4d config keyring-backend file
okp4d config node tcp://localhost:26657
okp4d init $NODENAME --chain-id okp4-nemeton

```
Создадим кошелек или восстановим существующий кошелек

```okp4d keys add ИМЯ_КОШЕЛЬКА```             #Чтобы создать новый (сохраняем сид фразу и задаем пароль)

``` okp4d keys add ИМЯ_КОШЕЛЬКА --recover ``` #Чтобы вернуть, используя сид фразу вашего кошелька



## Давайте установим Genesis и addrbook
```
wget -qO $HOME/.okp4d/config/genesis.json "https://raw.githubusercontent.com/okp4/networks/main/chains/nemeton/genesis.json"
```

## Давайте установим ПИРЫ
```
PEERS=f595a1386d5ca2e0d2cd81d3c6372c3bf84bbd16@65.109.31.114:2280,a49302f8999e5a953ebae431c4dde93479e17155@162.19.71.91:26656,dc14197ed45e84ca3afb5428eb04ea3097894d69@88.99.143.105:26656,79d179ea2e1fbdcc0c59a95ab7f1a0c48438a693@65.108.106.131:26706,501ad80236a5ac0d37aafa934c6ec69554ce7205@89.149.218.20:26656,5fbddca54548bf125ee96bb388610fe1206f087f@51.159.66.123:26656,769f74d3bb149216d0ab771d7767bd39585bc027@185.196.21.99:26656,024a57c0bb6d868186b6f627773bf427ec441ab5@65.108.2.41:36656,fff0a8c202befd9459ff93783a0e7756da305fe3@38.242.150.63:16656,2bfd405e8f0f176428e2127f98b5ec53164ae1f0@142.132.149.118:26656,bf5802cfd8688e84ac9a8358a090e99b5b769047@135.181.176.109:53656,dc9a10f2589dd9cb37918ba561e6280a3ba81b76@54.244.24.231:26656,085cf43f463fe477e6198da0108b0ab08c70c8ab@65.108.75.237:6040,803422dc38606dd62017d433e4cbbd65edd6089d@51.15.143.254:26656,b8330b2cb0b6d6d8751341753386afce9472bac7@89.163.208.12:26656

sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.okp4d/config/config.toml
```

## Давайте настроим обрезку
```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="50"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.okp4d/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.okp4d/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.okp4d/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.okp4d/config/app.toml
```

## Сбросим данные цепочки
```
okp4d tendermint unsafe-reset-all --home $HOME/.okp4d
```


## Давайте создадим сервис
```
sudo tee /etc/systemd/system/okp4d.service > /dev/null <<EOF
[Unit]
Description=okp4d
After=network-online.target

[Service]
User=$USER
ExecStart=$(which okp4d) start --home $HOME/.okp4d
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Давайте запустим узел
```
sudo systemctl daemon-reload
sudo systemctl enable okp4d
sudo systemctl restart okp4d && sudo journalctl -u okp4d -f -o cat
```

Давайте создадим валидатор (Но сначала ждем пока нода синхронизируется)


 Запросим тестовые токены с крана https://faucet.okp4.network/

Адрес кошелька смотрим при его создании

Текущую высоту можно посмотреть в эксплоере.

https://explorer.bccnodes.com/okp4

!!! Замените имя кошелька и моникера на СВОЙ

```
okp4d tx staking create-validator \
  --amount 1000000uknow \
  --from WALLETNAME \
  --commission-max-change-rate "0.01" \
  --commission-max-rate "0.2" \
  --commission-rate "0.05" \
  --min-self-delegation "1" \
  --pubkey  $(okp4d tendermint show-validator) \
  --moniker NODENAME \
  --chain-id okp4-nemeton
```

### Полезные команды
# Проверить логи
```
journalctl -fu okp4d -o cat
```

# запустить службу
```
sudo systemctl start okp4d
```

# остановить службу
```
sudo systemctl stop okp4d
```

# Перезапустите службу

```
sudo systemctl restart okp4d
```
# делегировать валидатору

# пример команды. подставьте ниже свои значения, в примере 2 токена
# okp4d tx staking delegate okp4valoper1xckd3h5cmkr2rhxd0q9tuntthvlkew2kcfhd6z 2000000uknow --from otmorozky-nemeton --chain-id=okp4-nemeton --fees 555uknow -y

```
okp4d tx staking delegate ВАШ_ВАЛОПЕРАДРЕС 2000000uknow --from=имя_кошелька --chain-id=okp4-nemeton --fees 555uknow -y
```

# делегировать от одного валидатора другому валидатору
```
okp4d tx staking redelegate адрес_валидатора адрес_валидатора 1000000uknow --from имя_кошелька --chain-id=okp4-nemeton --fees 555uknow -y
```

# Проверить баланс (будет отображаться только после полного синхрона)

```
okp4d q bank balances адрес-кошелька
# amount: "1000000"
```

# Дождитесь полной синхронизации ноды.
# Проверить можно

```
curl -s localhost:26657/status
# Нода синхронизирована, если в строчке "catching_up" значение false
```
# установка avatar (инструкция можно посмотреть здесь https://teletype.in/@letskynode/Archway_RU#ApGa )


```
okp4d tx staking edit-validator \
 --identity "код_с_сайта" \
 --details "Gruppa_Pervogo_Proekta" \
 --node `grep -oPm1 "(?<=^laddr = \")([^%]+)(?=\")" $HOME/.okp4d/config/config.toml` \
 --from "имя_кошелька"
```

# выход из тюрьмы
```
okp4d tx slashing unjail --from имя_кошелька --fees 555uknow -y
```
# Голосование
```
okp4d tx gov vote 1 yes --from имя_кошелька --chain-id=okp4-nemeton
```
#  Вывести все награды:
```
okp4d tx distribution withdraw-all-rewards --from имя_кошелька --chain-id=okp4-nemeton --fees 555uknow -y
```

## Удаление ноды

```
sudo systemctl stop okp4d
sudo systemctl disable okp4d
sudo rm /etc/systemd/system/okp4* -rf
sudo rm $(which okp4d) -rf
sudo rm $HOME/.okp4d* -rf
sudo rm $HOME/okp4d -rf
sed -i '/OKP_/d' ~/.bash_profile
```
