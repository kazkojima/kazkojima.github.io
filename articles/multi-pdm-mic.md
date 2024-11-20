Title: Multiple PDM mics with Raspberry Pi pico
Date: 2021-11-04
Save_as:multi-pdm-mic.html
Tags: acoustic
Slug: multi-pdm-mic

## 複数 PDM マイク出力を平均化して USB でアクセス

Raspberry Pi Pico で PDM マイクを USBマイク化できる [Microphone Library for Pico](https://github.com/ArmDeveloperEcosystem/microphone-library-for-pico.git)を複数マイクの平均を取得できるように改造してみました。

PDM マイクの信号は与えた clock 信号のタイミングでマイクの data 出力が 1 となる確率つまり 1 の密度で信号の振幅が決まります。上記のライブラリではこのタイミングで PIO の in 命令でマイクの data 出力を読み込んで PIO の入力シフトレジスタに 1-bit 入力しつつ 1-bit シフトします。これが 8-bit たまったところで PIO の FIFO に転送し、さらに DMA で メモリに転送、という処理を行っています。

例えば２つのマイクがあった時




