## 安装 s-ui 一键脚本

```bash
bash <(curl -Ls https://raw.githubusercontent.com/kiryusento2017/demo/main/s-ui-install.sh)
```
## Reality 延迟测试

<div id="container">
  <div id="buttonBar">
    <button id="refreshBtn">换一批</button>
    <button id="copyBtn">复制</button>
  </div>

  <textarea id="commandBox" readonly></textarea>
</div>

<style>
#container {
  position: relative;
  width: 100%;
}

#buttonBar {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-bottom: 8px;
}

#commandBox {
  width: 100%;
  height: 120px;
  font-family: monospace;
  font-size: 14px;
  padding: 10px;
  box-sizing: border-box;
  resize: none;
}
</style>

<script>
const domains = [
  "amd.com",
  "aws.com",
  "intel.com",
  "www.apple.com",
  "www.microsoft.com",
  "www.nvidia.com",
  "www.xbox.com",
  "www.oracle.com"
];

function getRandomDomains(array, count) {
  const shuffled = [...array].sort(() => 0.5 - Math.random());
  return shuffled.slice(0, count);
}

function generateShellCommand(selectedDomains) {
  const domainStr = selectedDomains.join(" ");

  return `for d in ${domainStr}; do
t1=$(date +%s%3N)
timeout 1 openssl s_client -connect $d:443 -servername $d </dev/null &>/dev/null &&
t2=$(date +%s%3N) &&
echo "$d: $((t2 - t1)) ms" ||
echo "$d: timeout"
done`;
}

function updateCommand() {
  const selected = getRandomDomains(domains, 5);
  const command = generateShellCommand(selected);

  document.getElementById("commandBox").value = command;
}

function copyToClipboard() {
  const textArea = document.getElementById("commandBox");

  textArea.select();
  document.execCommand("copy");
}

document.addEventListener("DOMContentLoaded", () => {
  updateCommand();

  document
    .getElementById("refreshBtn")
    .addEventListener("click", updateCommand);

  document
    .getElementById("copyBtn")
    .addEventListener("click", copyToClipboard);
});
</script>
