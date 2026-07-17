---
layout: post
title: SD卡极限压测与驱动死机崩溃排查记录
description: 记一次在 SigmaStar 平台上通过高压写入压测 SD 卡，导致底层驱动报错 Cmd_25 超时与 I/O Error 的原因排查及优化建议
summary: 记一次在 SigmaStar 平台上通过高压写入压测 SD 卡，导致底层驱动报错 Cmd_25 超时与 I/O Error 的原因排查及优化建议
tags: [doc, linux, driver, sdmmc]
---

### 压测背景与测试程序设计

*   **双线程异步架构**：设计了生产线程专门负责产生数据，写入线程专门负责写盘，中间通过有界队列（最大长度限制为2）进行通信，强制生产端与写入端同步节奏，防止内存溢出 (OOM)。
*   **极限防压缩机制**：使用 `xorshift32` 快速伪随机算法填充内存，彻底废掉 SD 卡内部主控的压缩策略（FTL），使得每一笔数据都能引发真实的 NAND Flash 物理擦写。
*   **连续高压写入**：每个测试文件由一个 6MB 的首段基础数据块和 1~50 个随机大小（1KB-50MB）的附加数据块组成。更严苛的是，压测启用了强制缓存刷盘（`fdatasync`），要求所有写入数据立刻绕过系统缓存直接落盘。

### 压测现象与内核报错

开启强制刷盘进行测试后，刚完成极少量写入，程序便卡死，随后内核打出致命错误。通过分析相关的底层日志，观察到以下现象：
*   底层驱动频繁刷屏警告：`Warn: #Cmd_25 (0x00...)=>(E: 0x0002)`。
*   伴随出现无尽的信号相位调优动作打印：`SDR PH(11), $$ PassPHs...`。
*   最终系统抛出块设备级别的致命异常：`blk_update_request: I/O error, dev mmcblk0, sector ... op 0x1:(WRITE)`。

### 崩溃原因深度剖析

结合系统底层的驱动源码，分析出此次死机崩溃是由软硬两方面的三大瓶颈共同导致的：

*   **SD卡硬件瓶颈（SLC Cache击穿）**：高频、不可压缩的超大块随机数据写入瞬间耗尽了廉价 SD 卡的内部缓存。卡片被迫触发后台垃圾回收 (GC)，导致底层的多块写指令（Cmd 25: WRITE_MULTIPLE_BLOCK）响应出现秒级的剧烈延迟。
*   **驱动超时设定过于苛刻**：在底层的配置文件中，控制硬件响应等待的超时阈值设定得过短（仅为 1000 毫秒）。当卡片处于内部 GC 忙碌状态超过 1 秒时，驱动便失去耐心，强行判定通讯超时，从而抛出 `E: 0x0002` 错误。
*   **致命的“相位重调”陷阱**：在底层错误捕获逻辑中，一旦发生上述通讯超时，驱动竟然武断地判定这是高速信号发生了相位偏移，随后强行触发 `Scan SDR Phase` 流程。此时 SD 卡其实只是单纯忙于写盘，主控却对着忙碌的卡疯狂重新校准 SDR 时钟相位，最终导致时序全盘崩溃，抛出无法挽回的 `I/O Error`。

### 改进与优化建议

基于上述排查结果，为了提高系统在极限情况下的鲁棒性，建议采取以下详细的优化措施：

**1. 修正底层驱动超时与错误恢复逻辑（治本）**
*   **放宽硬件超时阈值**：定位到驱动配置头文件（例如 `hal_card_platform_config.h`），将其中的宏定义 `WT_EVENT_RSP` 由默认的 `1000` 放宽至 `3000` 或 `5000` 毫秒。这能给底层便宜的 SD 卡留出充足的后台 GC 搬运数据的时间，避免因短暂的卡顿引发严重的内核断联。
*   **规避致命的相位陷阱**：在底层错误分发层文件 `ms_sdmmc_lnx.c` 中（定位到包含 `Scan SDR Phase` 注释的 1385 行附近逻辑），必须增加对错误代码 `eErr` 的精确拦截。当 `eErr == EV_STS_MIE_TOUT`（纯粹的忙碌/响应超时）时，应当直接 `return` 报错或发起静默重传，**严格禁止**代码继续往下走到重调相位的流程。只有在明确捕获到 `EV_STS_RSP_CERR`（发生真正的物理 CRC 校验错）时，才允许启动这套耗时的相位扫描逻辑。

**2. 修改设备树 (DTS)，降频并增强驱动能力（最快捷解法）**
*   **改动点**：定位到平台的设备树文件（例如 `infinity6f.dtsi`）中的 `sdmmc` 节点。
*   **关闭高速模式 (SD 3.0)**：将 `slot-support-sd30 = <1>,<0>,<0>;` 修改为 `<0>,<0>,<0>;`。这会从硬件协商阶段直接禁用 UHS-I 超高速模式，迫使 SD 卡回退到 50MHz 的传统 High-Speed 模式。在此模式下，内核驱动**根本不会执行**那段充满陷阱的 SDR 相位扫描逻辑，系统稳定性将呈指数级提升。同时也强烈建议将 `slot-max-clks` 的最高频 200MHz 降频至 50MHz。
*   **适度增强信号驱动能力 (Driving Strength)**：如果由于主板走线过长或寄生电容过大导致高频信号波形畸变，可以在该节点下找到如下属性：
    ```dts
    slot-clk-driving  = <2>,<2>,<0>;
    slot-cmd-driving  = <2>,<2>,<0>;
    slot-data-driving = <2>,<2>,<0>;
    ```
    将表示外置 SD 卡槽（通常是第 0 个插槽的数值）的驱动能力档位从默认的 `<2>` 适度提升至 `<3>` 或 `<4>`（档位最大值为 7）。以此来提升物理层时钟与数据线边沿的陡峭程度，改善信号眼图。但请注意，盲目拉高驱动能力可能会引入电磁干扰 (EMI) 和过冲问题，建议配合示波器监测微调。

**3. 应用层写盘平滑化调优 (sysctl 参数配置)**
*   **优化目标**：在持续录像等业务中应禁用业务层的强制 `sync`，转而依靠 Linux 内核管理脏页（Dirty Pages）回写。然而，内核默认配置倾向于在内存中“攒一大波数据”再一口气刷盘，极易造成瞬时的巨量 I/O 爆发（波峰式挤压），直接冲垮廉价 SD 卡的内部缓冲。我们需要通过调整让内核“少吃多餐”，平滑且及早地往 SD 卡写数据。
*   **需要调整的核心参数与建议值**：
    *   `vm.dirty_background_ratio`：触发后台内核线程异步刷盘的脏页占总内存的百分比。建议从默认的 10 调低至 `2` ~ `5`，让系统在只积攒了极少量脏数据时就立刻开始慢慢向后台写盘。
    *   `vm.dirty_ratio`：触发所有产生写入动作的进程同步阻塞（强制交出 CPU 去刷盘）的硬上限百分比。建议从默认的 20 调低至 `10` 左右，以压低极限缓冲的危险水位。
    *   `vm.dirty_writeback_centisecs`：后台刷盘守护进程的唤醒巡视间隔（单位：百分之一秒，100 = 1秒）。建议从默认的 500 调低至 `100` 或 `50`，让内核更勤快地去搬运脏数据。
    *   `vm.dirty_expire_centisecs`：脏数据在内存中停留的最长允许老化时间（单位：百分之一秒）。建议从默认的 3000 (30秒) 调低至 `500` (5秒) 或 `1000` (10秒)。
*   **如何修改**：
    *   **临时生效（热修改调参测试）**：通过终端直接下发命令：
        ```bash
        sysctl -w vm.dirty_background_ratio=5
        sysctl -w vm.dirty_ratio=10
        sysctl -w vm.dirty_writeback_centisecs=100
        sysctl -w vm.dirty_expire_centisecs=500
        ```
        *(或者通过 `echo 5 > /proc/sys/vm/dirty_background_ratio` 方式写入)*
    *   **永久生效（固化入固件）**：将参数写入设备根文件系统的 `/etc/sysctl.conf` 文件中：
        ```ini
        vm.dirty_background_ratio = 5
        vm.dirty_ratio = 10
        vm.dirty_writeback_centisecs = 100
        vm.dirty_expire_centisecs = 500
        ```
        确保系统的 `init` 或 `rc.local` 启动脚本中带有 `sysctl -p` 指令，从而在每次开机时自动载入。

**4. 硬件供电排查（硬件层）**
*   **排查重点**：在执行高压满载写入的瞬间，通过示波器抓取 `VDD_SD` 引脚的电压波形。
*   **修改方案**：排查高压写入带来的瞬时大电流是否导致了 LDO 供电出现了纹波跌落。如确认存在压降，需考虑在电源侧增加滤波电容或更换更大带载能力的供电芯片。

### 补充：实际应用中是否建议使用 Sync？

**结论：在 IPC 监控摄像机等高频写盘场景中，极其不推荐高频或逐块使用 `sync` / `fdatasync`。**

压测中使用强制 `sync` 是为了刺探驱动底线的**极端测试手段**，但在实际落地业务中，滥用 `sync` 会带来灾难性后果：

1. **引发严重的“写放大”(Write Amplification)**：SD 卡内部的 NAND Flash 只能按块（Block，通常为 4MB 甚至更大）进行擦写。如果您每写入几十 KB 数据就调用一次 `sync` 强刷，SD 卡将被迫反复搬运、擦除并重写同一个物理大块。这会导致卡片读写性能断崖式暴跌，且极易引发超时卡死。
2. **极速损耗 SD 卡寿命**：消费级 SD 卡的 TLC/QLC 颗粒擦写寿命本身就十分有限（通常仅有几百到一千次 P/E）。高频强制落盘会迅速耗尽主控的磨损均衡（Wear Leveling）裕量，导致一张原本寿命达数年的卡在短短几周甚至几天内直接写废“变砖”。
3. **正确的数据落盘姿势**：
   * **信任系统缓冲**：绝大部分视音频流数据应当交由 Linux 内核的 Page Cache 脏页回写机制去处理，系统会自动将零碎数据聚合成大块后再一次性写给底层，大大降低 SD 卡 FTL 的负担。
   * **业务对齐刷盘**：如果确有如“录像切片索引表”这样防丢的关键元数据，应当在内存中先“攒大块”（例如按 1MB 边界对齐），随后再低频地调用一次 `fsync()`，切忌“碎文件强刷”。
   * **硬件级掉电保护**：如果是为了防止突发断电导致最后几秒录像丢失，更正统的解法是在硬件端加入超级电容（SuperCap）储能方案，在检测到外部掉电中断的最后几百毫秒内，再去执行一次总的 `sync` 安全收尾。

### 附录：压测工具源码 (`sd_stress.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <fcntl.h>
#include <time.h>
#include <sys/stat.h>
#include <sys/mount.h>
#include <errno.h>

#define FILE_F "/tmp/test_6M.bin"
#define FILE_F_SIZE (6 * 1024 * 1024)

typedef struct Task {
    void *data;
    size_t size;
    struct timespec gen_start; // 数据开始生成时间
    struct timespec gen_end;   // 数据生成完毕，准备入队时间
    int loop_idx;
    int chunk_idx;
    int total_chunks;
    struct Task *next;
} Task;

Task *queue_head = NULL;
Task *queue_tail = NULL;
int queue_size = 0;
#define MAX_QUEUE_SIZE 2

pthread_mutex_t queue_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t queue_cond = PTHREAD_COND_INITIALIZER;
pthread_cond_t queue_not_full = PTHREAD_COND_INITIALIZER;

void *file_f_data = NULL;
char sd_dir[256] = "/mnt/sdcard";
int enable_sync = 0; // 同步开关
int max_loops = 0;   // 最大循环次数 (0表示无限)



// 获取时间戳之差（毫秒）
double diff_ms(struct timespec start, struct timespec end) {
    return (end.tv_sec - start.tv_sec) * 1000.0 + (end.tv_nsec - start.tv_nsec) / 1000000.0;
}

void enqueue(void *data, size_t size, struct timespec *t_start, struct timespec *t_end, int l_idx, int c_idx, int t_chunks) {
    Task *t = malloc(sizeof(Task));
    t->data = data;
    t->size = size;
    if (t_start) t->gen_start = *t_start;
    if (t_end) t->gen_end = *t_end;
    t->loop_idx = l_idx;
    t->chunk_idx = c_idx;
    t->total_chunks = t_chunks;
    t->next = NULL;

    pthread_mutex_lock(&queue_mutex);
    while (queue_size >= MAX_QUEUE_SIZE) {
        pthread_cond_wait(&queue_not_full, &queue_mutex);
    }

    if (queue_tail) {
        queue_tail->next = t;
        queue_tail = t;
    } else {
        queue_head = queue_tail = t;
    }
    queue_size++;
    pthread_cond_signal(&queue_cond);
    pthread_mutex_unlock(&queue_mutex);
}

Task *dequeue() {
    pthread_mutex_lock(&queue_mutex);
    while (queue_size == 0) {
        pthread_cond_wait(&queue_cond, &queue_mutex);
    }
    Task *t = queue_head;
    queue_head = t->next;
    if (queue_head == NULL) queue_tail = NULL;
    queue_size--;
    pthread_cond_signal(&queue_not_full); // 通知M队列有空位了
    pthread_mutex_unlock(&queue_mutex);
    return t;
}

// 快速伪随机数生成器（Xorshift32），比标准 rand() 快非常多
static inline unsigned int xorshift32(unsigned int *state) {
    unsigned int x = *state;
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    *state = x;
    return x;
}

// 使用快速随机算法填充整个 buffer
void fill_random_bytes(void *buf, size_t size, unsigned int *seed) {
    unsigned int *ptr = (unsigned int *)buf;
    size_t words = size / sizeof(unsigned int);
    for (size_t i = 0; i < words; i++) {
        ptr[i] = xorshift32(seed);
    }
    unsigned char *cptr = (unsigned char *)(ptr + words);
    size_t rem = size % sizeof(unsigned int);
    if (rem > 0) {
        unsigned int tail = xorshift32(seed);
        for (size_t i = 0; i < rem; i++) {
            cptr[i] = (tail >> (i * 8)) & 0xFF;
        }
    }
}

void init_file_f() {
    file_f_data = malloc(FILE_F_SIZE);
    unsigned int seed = 123456789;
    fill_random_bytes(file_f_data, FILE_F_SIZE, &seed);
    
    int fd = open(FILE_F, O_CREAT | O_WRONLY | O_TRUNC, 0644);
    if (fd < 0) {
        perror("Create F failed");
        exit(1);
    }
    write(fd, file_f_data, FILE_F_SIZE);
    close(fd);
    printf("[Init] F file created: %s (6MB)\n", FILE_F);
}

void *thread_m(void *arg) {
    srand(time(NULL));
    struct timespec t_f_ts;
    int current_loop = 0;
    
    while (max_loops == 0 || current_loop < max_loops) {
        current_loop++;
        
        // 2. 随机产生1KB-50MB数据 1-50次 (决定本轮的总块数)
        int repeat = (rand() % 50) + 1; // 1-50
        
        // 1. 每次循环开头，先将F送给W。F是预生成的，所以生成耗时算0
        clock_gettime(CLOCK_MONOTONIC, &t_f_ts);
        enqueue(file_f_data, FILE_F_SIZE, &t_f_ts, &t_f_ts, current_loop, 0, repeat);

        for (int i = 0; i < repeat; i++) {
            size_t size = (rand() % (50 * 1024 * 1024 - 1024 + 1)) + 1024;
            
            struct timespec t1, t2;
            clock_gettime(CLOCK_MONOTONIC, &t1); // 记录生成开始时间
            
            void *data = malloc(size);
            if (data) {
                unsigned int seed = rand();
                fill_random_bytes(data, size, &seed);
                clock_gettime(CLOCK_MONOTONIC, &t2); // 记录生成完毕时间
                enqueue(data, size, &t1, &t2, current_loop, i + 1, repeat);
            }
        }

        // 3. 随机等待 0-1000ms
        int wait_ms = rand() % 1001;
        usleep(wait_ms * 1000);
    }
    
    // 发送退出信号给 W
    enqueue(NULL, 0, NULL, NULL, 0, 0, 0);
    return NULL;
}

void *thread_w(void *arg) {
    char file_path[512];
    unsigned int file_idx = 0;
    int fd = -1;

    while (1) {
        Task *t = dequeue();
        
        // 收到退出信号
        if (t->data == NULL && t->size == 0) {
            free(t);
            break;
        }

        int is_full = 0;

        // [F] 数据，作为新文件起始
        if (t->data == file_f_data) {
            if (fd >= 0) {
                close(fd);
                fd = -1;
            }
            snprintf(file_path, sizeof(file_path), "%s/stress_%u.bin", sd_dir, file_idx);
            fd = open(file_path, O_CREAT | O_WRONLY | O_TRUNC, 0644);
            if (fd < 0) {
                perror("Open SD file err");
                if (errno == ENOSPC) is_full = 1;
            } else {
                file_idx++;
            }
        }

        // 开始写入前的时间戳
        struct timespec w_start, w_end, sync_end;
        clock_gettime(CLOCK_MONOTONIC, &w_start);

        if (fd >= 0) {
            ssize_t ret = write(fd, t->data, t->size);
            clock_gettime(CLOCK_MONOTONIC, &w_end); // 写入系统缓存完成后的时间戳

            double sync_ms = 0.0;
            if (ret >= 0 && enable_sync) {
                fdatasync(fd);
                clock_gettime(CLOCK_MONOTONIC, &sync_end);
                sync_ms = diff_ms(w_end, sync_end);
            }

            if (ret < 0 || (size_t)ret < t->size) {
                perror("Write err / Disk full");
                is_full = 1;
            } else {
                double m_gen_ms = diff_ms(t->gen_start, t->gen_end);
                double wait_ms = diff_ms(t->gen_end, w_start);
                double w_ms = diff_ms(w_start, w_end);
                
                if (enable_sync) {
                    printf("[W] [Lp %d:%d/%d] Appended %zu B to %s | M_Gen: %.3f ms | Wait: %.3f ms | W(Cache): %.3f ms | Sync: %.3f ms\n", 
                            t->loop_idx, t->chunk_idx, t->total_chunks, t->size, file_path, m_gen_ms, wait_ms, w_ms, sync_ms);
                } else {
                    printf("[W] [Lp %d:%d/%d] Appended %zu B to %s | M_Gen: %.3f ms | Wait: %.3f ms | W_Time: %.3f ms\n", 
                            t->loop_idx, t->chunk_idx, t->total_chunks, t->size, file_path, m_gen_ms, wait_ms, w_ms);
                }
            }
        }

        if (is_full) {
            printf("[W] SD Card likely full. Deleting all test files...\n");
            if (fd >= 0) {
                close(fd);
                fd = -1;
            }
            char cmd[512];
            snprintf(cmd, sizeof(cmd), "rm -f %s/stress_*.bin", sd_dir);
            system(cmd);
            file_idx = 0;
        }

        if (t->data != file_f_data) {
            free(t->data);
        }
        free(t);
    }
    
    if (fd >= 0) close(fd);
    printf("[W] Exit signal received. Thread W stopped.\n");
    return NULL;
}

int main(int argc, char **argv) {
    char *dev = "/dev/mmcblk0p1";
    if (argc > 1) dev = argv[1];
    if (argc > 2) strncpy(sd_dir, argv[2], sizeof(sd_dir) - 1);
    if (argc > 3) enable_sync = atoi(argv[3]);
    if (argc > 4) max_loops = atoi(argv[4]);

    printf("--- SD Stress Test ---\n");
    printf("Device: %s\n", dev);
    printf("Mount Dir: %s\n", sd_dir);
    printf("Force Sync: %s\n", enable_sync ? "ON" : "OFF");
    printf("Max Loops: %d (0 = Infinite)\n", max_loops);
    printf("----------------------\n");

    mkdir(sd_dir, 0755);
    if (mount(dev, sd_dir, "vfat", 0, NULL) == 0) {
        printf("[Init] Mounted %s to %s\n", dev, sd_dir);
    } else {
        perror("[Init] Mount warning");
    }

    init_file_f();

    pthread_t tm, tw;
    pthread_create(&tm, NULL, thread_m, NULL);
    pthread_create(&tw, NULL, thread_w, NULL);

    pthread_join(tm, NULL);
    pthread_join(tw, NULL);
    
    return 0;
}
```
