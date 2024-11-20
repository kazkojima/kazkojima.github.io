Title: Jetson nanoでCUDAnative.jl
Date: 2020-05-16
Save_as: cuda-jl.html
Tags: Julia, CUDA, memo
Slug: cuda-jl

## Jetson nanoでCUDAnative.jlのサンプルを実行

# Juliaのインストール

* https://julialang.org/downloads/からGeneric Linux Binaries for ARMの64-bit(AArch64)用のJulia(例えばjulia-1.4.1-linux-aarch64.tar.gz)をdownload
* それを例えば/optに展開し/opt/julia-1.4.1/binをPATHに追加

# Juliaで必要なパッケージを追加

* Juliaを起動してpkgモードで
```
add CUDAnative
add CUDAdrv
add CuArrays
```

# Sampleを走らせる
```
# listing 16
using CUDAnative, CuArrays

function vadd(a, b, c)
    i = (blockIdx().x-1)*blockDim().x + threadIdx().x
    c[i] = a[i] + b[i]
    return
end

len = 100
a = rand(Float32, len)
b = rand(Float32, len)

d_a = CuArray(a)
d_b = CuArray(b)
d_c = similar(d_a)

@time @cuda threads=len vadd(d_a, d_b, d_c)
```
これはcompileするので時間がかかります。Jetson nanoだと
  49.464224 seconds (32.68 M allocations: 1.546 GiB, 4.34% gc time)

ここで
```
c = Base.Array(d_c)
```
で結果を見れます。２回めの呼び出しは速いです。

```
ulia> @time @cuda threads=len vadd(d_a, d_b, d_c)
  0.016108 seconds (65 allocations: 2.141 KiB)
```

## CUDAnative.jl メモ

JuliaでCUDAのkernel関数を記述して呼ぶことが出来る[CUDAnative.jl](https://github.com/JuliaGPU/CUDAapi.jl.git)を読もうとしての[メモ]({filename}/images/cudanative-note.pdf)
