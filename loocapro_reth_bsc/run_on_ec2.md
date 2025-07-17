# run a full node in ec2
```sh
tsh login --proxy=tele-bnbchain.iac.toolsfdg.net:8024 bnbchain --auth=tf-okta_connector_nodereal

tsh ssh clyde.m@10.179.43.46

sudo su


df -h
文件系统        容量  已用  可用 已用% 挂载点
devtmpfs         16G     0   16G    0% /dev
tmpfs            16G     0   16G    0% /dev/shm
tmpfs            16G  516K   16G    1% /run
tmpfs            16G     0   16G    0% /sys/fs/cgroup
/dev/nvme0n1p1   60G  6.1G   54G   11% /
/dev/nvme1n1    3.0T   21G  3.0T    1% /server
tmpfs           3.1G     0  3.1G    0% /run/user/0

# install rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh


```

## cmd
```sh
tmux new -s reth-bsc
#Split vertically
Ctrl-b %
#Split horizontally:
Ctrl-b "        #"

# Move between panes with arrow keys:
Ctrl-b ↑
Ctrl-b ↓
Ctrl-b ←
Ctrl-b →

# detach
Ctrl-b d

# Maximizing" a pane in tmux
Ctrl + b + z  # toggle maximizing


c  create window
,  name window
0-9 go to the giving N-th number window






cd /server/workdir/rust_for_bsc/reth-bsc 

sudo yum install -y clang clang-devel llvm-devel
sudo yum groupinstall "Development Tools"

mkdir -p /server/clyde.m/.local/share/reth
mkdir -p /server/clyde.m/.cache/reth/logs

RUSTFLAGS='-C link-arg=-lgcc' cargo build --bin reth-bsc --release

# 500w 0xc47ffcf02770b9510df200e30811e915772770658cac69af9bd40df97dba1cb6
# start 2025-07-02T11:49:01.968808Z
# finish 2025-07-02T17:37:04.024218Z


# 1500w 0xf5e5f5655df562161055fbdaaa4986b3e22cd327121af7eef2be2b115efb05be
# 2025-07-02T20:59:36.931374Z  start
# 2025-07-04T01:00:09.188895Z now




RUST_LOG=info   ./target/release/reth-bsc node --chain bsc --http --http.api="eth, net, txpool, web3, rpc" --metrics 0.0.0.0:9001  --datadir /server/clyde.m/.local/share/reth --log.file.directory /server/clyde.m/.cache/reth/logs --debug.tip 0xf5e5f5655df562161055fbdaaa4986b3e22cd327121af7eef2be2b115efb05be 

```
## other

```sh
curl http://localhost:9001/metrics > metrics.txt

ps -f 30495
```

## 
0. 性能评测(Clyde, Dike)
   - block level 性能  mgasps  ， 追块速度
   - execution ， 访问state, 访问memory的metrics
1. snapshot没有办法支持 (Joey)
2. 社区目前支持到 hetz hardfork  （社区在支持，看看社区支持的怎么样。我们不作为高优先级）
3. zoro prs (Fynn) Clyde/Dike
4. merkel tree (depends on 0)
5. super instruction