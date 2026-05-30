#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <dirent.h>

#define MAX_CHAR 256
#define PATH_MAX 260

// 霍夫曼树节点
typedef struct HuffNode {
    unsigned char ch;
    unsigned int freq;
    struct HuffNode *left, *right;
} HuffNode;

// 编码表
typedef struct HuffCode {
    unsigned char bits[256];
    int len;
} HuffCode;

HuffCode codeTable[MAX_CHAR];

// 文件头
typedef struct FileHeader {
    char fileName[256];
    unsigned int fileSize;
} FileHeader;

// 清空编码表
void clearCodeTable() {
    memset(codeTable, 0, sizeof(codeTable));
}

// 释放霍夫曼树
void freeHuffTree(HuffNode *root) {
    if (!root) return;
    freeHuffTree(root->left);
    freeHuffTree(root->right);
    free(root);
}

// 创建节点
HuffNode* createNode(unsigned char ch, unsigned int freq) {
    HuffNode *node = (HuffNode*)malloc(sizeof(HuffNode));
    node->ch = ch;
    node->freq = freq;
    node->left = node->right = NULL;
    return node;
}

// 排序
void sortNodes(HuffNode **nodes, int n) {
    int i, j;
    for (i = 0; i < n - 1; i++) {
        for (j = i + 1; j < n; j++) {
            if (nodes[i]->freq > nodes[j]->freq) {
                HuffNode *temp = nodes[i];
                nodes[i] = nodes[j];
                nodes[j] = temp;
            }
        }
    }
}

// 构建霍夫曼树
HuffNode* buildHuffTree(unsigned int freq[]) {
    HuffNode **nodes, *root;
    int n = 0, i;
    for (i = 0; i < MAX_CHAR; i++) if (freq[i] > 0) n++;
    nodes = (HuffNode**)malloc(n * sizeof(HuffNode*));

    int idx = 0;
    for (i = 0; i < MAX_CHAR; i++) {
        if (freq[i] > 0) nodes[idx++] = createNode((unsigned char)i, freq[i]);
    }

    while (n > 1) {
        sortNodes(nodes, n);
        HuffNode *l = nodes[0], *r = nodes[1];
        HuffNode *p = createNode(0, l->freq + r->freq);
        p->left = l; p->right = r;
        for (i = 2; i < n; i++) nodes[i - 2] = nodes[i];
        nodes[n - 2] = p;
        n--;
    }
    root = nodes[0];
    free(nodes);
    return root;
}

// 生成编码
void generateCodes(HuffNode *root, unsigned char bits[], int len) {
    if (!root) return;
    if (!root->left && !root->right) {
        int i;
        codeTable[(unsigned char)root->ch].len = len;
        for (i = 0; i < len; i++) codeTable[(unsigned char)root->ch].bits[i] = bits[i];
        return;
    }
    bits[len] = 0; generateCodes(root->left, bits, len + 1);
    bits[len] = 1; generateCodes(root->right, bits, len + 1);
}

// 统计频率
void getFreq(FILE *fp, unsigned int freq[]) {
    unsigned char ch;
    while (fread(&ch, 1, 1, fp) == 1) freq[ch]++;
    fseek(fp, 0, SEEK_SET);
}

// 获取文件大小
long getFileSize(const char *path) {
    FILE *f = fopen(path, "rb");
    if(!f) return 0;
    fseek(f, 0, SEEK_END);
    long s = ftell(f);
    fclose(f);
    return s;
}

// 单文件压缩
void compress(const char *in, const char *out) {
    FILE *fin = fopen(in, "rb");
    if (!fin) { printf("未找到文件：%s\n", in); exit(1); }
    FILE *fout = fopen(out, "wb");
    unsigned int freq[MAX_CHAR] = {0};
    unsigned char buf = 0;
    int bitCnt = 0;
    unsigned char tail_bits = 0;

    getFreq(fin, freq);
    fwrite(freq, sizeof(unsigned int), MAX_CHAR, fout);
    fwrite(&tail_bits, 1, 1, fout); // 占位

    HuffNode *root = buildHuffTree(freq);
    unsigned char bits[256];
    
    clearCodeTable();
    generateCodes(root, bits, 0);

    unsigned char ch;
    while (fread(&ch, 1, 1, fin) == 1) {
        HuffCode c = codeTable[ch];
        for (int i = 0; i < c.len; i++) {
            buf = (buf << 1) | c.bits[i];
            bitCnt++;
            if (bitCnt == 8) {
                fwrite(&buf, 1, 1, fout);
                buf = 0; bitCnt = 0;
            }
        }
    }
    if (bitCnt > 0) {
        tail_bits = bitCnt;
        buf <<= (8 - bitCnt);
        fwrite(&buf, 1, 1, fout);
    }
    // 回写真实有效位数
    fseek(fout, MAX_CHAR * sizeof(unsigned int), SEEK_SET);
    fwrite(&tail_bits, 1, 1, fout);

    fclose(fin); fclose(fout);
    freeHuffTree(root);
}

//单文件解压
void decompress(const char *in, const char *out) {
    FILE *fin = fopen(in, "rb");
    if (!fin) { printf("未找到文件：%s\n", in); exit(1); }
    FILE *fout = fopen(out, "wb");
    unsigned int freq[MAX_CHAR] = {0};
    unsigned char tail_bits = 0;

    fread(freq, sizeof(unsigned int), MAX_CHAR, fin);
    fread(&tail_bits, 1, 1, fin);

    HuffNode *root = buildHuffTree(freq);
    HuffNode *curr = root;
    unsigned char ch;

    while (fread(&ch, 1, 1, fin) == 1) {
        int use = (feof(fin) && tail_bits != 0) ? tail_bits : 8;
        for (int i = 7; i >= 8 - use; i--) {
            int bit = (ch >> i) & 1;
            curr = (bit == 0) ? curr->left : curr->right;
            if (!curr->left && !curr->right) {
                fwrite(&curr->ch, 1, 1, fout);
                curr = root;
            }
        }
    }
    fclose(fin); fclose(fout);
    freeHuffTree(root);
}

// 多文件压缩
void compressMulti(char *files[], int fileCnt, const char *out) {
    FILE *fout = fopen(out, "wb");
    fwrite(&fileCnt, sizeof(int), 1, fout);

    for (int i = 0; i < fileCnt; i++) {
        char *fname = files[i];
        FILE *fin = fopen(fname, "rb");
        if (!fin) { printf("未找到文件：%s\n", fname); exit(1); }
        fseek(fin, 0, SEEK_END);
        unsigned int size = ftell(fin);
        fseek(fin, 0, SEEK_SET);

        FileHeader fh;
        strcpy(fh.fileName, fname);
        fh.fileSize = size;
        fwrite(&fh, sizeof(FileHeader), 1, fout);

        unsigned int freq[MAX_CHAR] = {0};
        getFreq(fin, freq);
        fwrite(freq, sizeof(unsigned int), MAX_CHAR, fout);
        unsigned char tail_bits = 0;
        fwrite(&tail_bits, 1, 1, fout); // 占位

        HuffNode *root = buildHuffTree(freq);
        unsigned char bits[256];
        
        clearCodeTable();
        generateCodes(root, bits, 0);
        fseek(fin, 0, SEEK_SET);

        unsigned char ch, buf = 0;
        int bitCnt = 0;
        while (fread(&ch, 1, 1, fin) == 1) {
            HuffCode c = codeTable[ch];
            for (int j = 0; j < c.len; j++) {
                buf = (buf << 1) | c.bits[j];
                bitCnt++;
                if (bitCnt == 8) {
                    fwrite(&buf, 1, 1, fout);
                    buf = 0; bitCnt = 0;
                }
            }
        }
        if (bitCnt > 0) {
            tail_bits = bitCnt;
            buf <<= (8 - bitCnt);
            fwrite(&buf, 1, 1, fout);
        }
        // 回写
        fseek(fout, -(1 + (bitCnt>0?1:0)), SEEK_CUR);
        fwrite(&tail_bits, 1, 1, fout);
        fseek(fout, 0, SEEK_END);

        fclose(fin);
        freeHuffTree(root);
    }
    fclose(fout);
}

// 多文件解压
void decompressMulti(const char *in) {
    FILE *fin = fopen(in, "rb");
    if (!fin) { printf("未找到文件：%s\n", in); exit(1); }
    int fileCnt;
    fread(&fileCnt, sizeof(int), 1, fin);

    for (int i = 0; i < fileCnt; i++) {
        FileHeader fh;
        fread(&fh, sizeof(FileHeader), 1, fin);
        FILE *fout = fopen(fh.fileName, "wb");

        unsigned int freq[MAX_CHAR] = {0};
        fread(freq, sizeof(unsigned int), MAX_CHAR, fin);
        unsigned char tail_bits = 0;
        fread(&tail_bits, 1, 1, fin);

        HuffNode *root = buildHuffTree(freq);
        HuffNode *curr = root;
        unsigned char ch;

        while (ftell(fout) < fh.fileSize) {
            fread(&ch, 1, 1, fin);
            int use = 8;
            if (ftell(fout) + 1 >= fh.fileSize && tail_bits != 0) use = tail_bits;

            for (int j = 7; j >= 8 - use; j--) {
                int bit = (ch >> j) & 1;
                curr = (bit == 0) ? curr->left : curr->right;
                if (!curr->left && !curr->right) {
                    fwrite(&curr->ch, 1, 1, fout);
                    curr = root;
                    if (ftell(fout) >= fh.fileSize) break;
                }
            }
        }
        fclose(fout);
        freeHuffTree(root);
    }
    fclose(fin);
}

// 显示帮助
void help() {
    printf("===== 霍夫曼压缩软件 =====\n");
    printf("单文件压缩: huffman -c 输入 输出\n");
    printf("单文件解压: huffman -d 输入 输出\n");
    printf("多文件压缩: huffman -m 输出 文件1 文件2 ...\n");
    printf("多文件解压: huffman -x 压缩包\n");
    printf("=========================\n");
}

int main(int argc, char *argv[]) {
    if (argc < 3) { help(); return 0; }
    clock_t start = clock();

    if (!strcmp(argv[1], "-c") && argc == 4) {
        compress(argv[2], argv[3]);
        long in_size = getFileSize(argv[2]);
        long out_size = getFileSize(argv[3]);
        double ratio = (double)out_size / in_size * 100;
        printf("【单文件压缩完成】\n");
        printf("输入文件大小：%ld 字节\n", in_size);
        printf("输出文件大小：%ld 字节\n", out_size);
        printf("压缩比：%.2f%%\n", ratio);
    } else if (!strcmp(argv[1], "-d") && argc == 4) {
        decompress(argv[2], argv[3]);
        printf("【单文件解压完成】\n");
    } else if (!strcmp(argv[1], "-m") && argc >= 4) {
        compressMulti(argv + 3, argc - 3, argv[2]);
        long total_in = 0;
        for (int i = 3; i < argc; i++) {
            total_in += getFileSize(argv[i]);
        }
        long out_size = getFileSize(argv[2]);
        double ratio = (double)out_size / total_in * 100;
        printf("【多文件压缩完成】\n");
        printf("所有输入文件总大小：%ld 字节\n", total_in);
        printf("压缩包文件大小：%ld 字节\n", out_size);
        printf("压缩比：%.2f%%\n", ratio);
    } else if (!strcmp(argv[1], "-x") && argc == 3) {
        decompressMulti(argv[2]);
        printf("【多文件解压完成】\n");
    } else { help(); return 0; }

    double time = (double)(clock() - start) / CLOCKS_PER_SEC;
    printf("总耗时：%.2f秒\n", time);
    return 0;
}
