---
title: "지표면의 고도를 나타내는 데이터를 입력 받아 이미지로 표현하는 프로그램"
last_modified_at: 2025-04-02T21:05:12+09:00
categories:
    - programmers
tags:
    - Flex
    - Bison
author_profile: true

---
# 문제 설명
지표면의 고도를 나타내는 데이터를 입력 받아 이미지로 표현하는 프로그램

입력 받은 데이터를 RGB 값으로 계산한 후 새로운 이미지 파일을 생성하여 이를 출력하는 프로그램

데이터 파일은 $m\times n$ 개의 숫자로 구성되어 있으며, 줄바꿈은 없다.

주어진 입력을 흑백 이미지로 출력하기 위해 입력된 데이터에서 최소값과 최대값을 찾고 shade of grey 계산

$shade of grey=\frac{elevation_{cur} - elevation_{min}}{elevation_{max} - elevation_{min}}\times 255$

계산한 값을 portable pixel map 형태로 저장. 흑백 이미지 이므로, 한 픽셀의 RGB 값은 동일

portable pixel map은 첫 번째 줄 문자열 P3, 두번째 줄에 width와 height, 세 번째 줄에 max color value, 나머지 줄에 RGB 값이 명시되어 있다.
```
P3
100 100
255
0 0 0 1 1 1 100 100 100 255 255 255
...
```
# 문제 풀이
```c
#include <stdio.h>
#include <math.h>
#include <stdlib.h>
int readFile(FILE *file, int **matrix, int rows, int cols) {
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (fscanf(file, "%d", &matrix[i][j]) != 1) {
                if (feof(file)) {
                    printf("Error: End of file reached prior to getting all the required data\n");
                } else {
                    printf("Error: Error while reading file.\n");
                }
            
                return 1; // Error reading the file
            }
        }
    }
    if (feof(file)) {
        printf("Error: End of file reached prior to getting all the required data\n");
        return 1; // Error reading the file
    }
    return 0; // Successfully read the file
}

int findMax(int **matrix, int rows, int cols) {
    int max = matrix[0][0];
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (matrix[i][j] > max) {
                max = matrix[i][j];
            }
        }
    }
    return max;
}

int findMin(int **matrix, int rows, int cols) {
    int min = matrix[0][0];
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (matrix[i][j] < min) {
                min = matrix[i][j];
            }
        }
    }
    return min;
}

void loadGreyScale(int **matrix, int rows, int cols) {
    int max = findMax(matrix, rows, cols);
    int min = findMin(matrix, rows, cols);
    for(int i=0;i<rows;++i){
        for(int j=0;j<cols;++j){
            
            matrix[i][j] = (int)round((float)(matrix[i][j] - min)*255 / (float)(max - min));
            
        }
    }
}

void saveFilePPM(char* originalFileName, int **matrix, int rows, int cols) {
    char filename[100];
    sprintf(filename,"%s.ppm", originalFileName);
    FILE *file = fopen(filename, "w");
    fprintf(file, "P3\n%d %d\n255\n", cols, rows);
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            fprintf(file, "%d %d %d ", matrix[i][j], matrix[i][j], matrix[i][j]);
        }
        fprintf(file, "\n");
    }
    fclose(file);
}

int main() {
    int rows, cols;
    char filename[100];
    int **matrix;
    FILE *file;

    // Prompt user for the number of rows and columns
    printf("Enter the number of rows: ");
    scanf("%d", &rows);
    printf("Enter the number of columns: ");
    scanf("%d", &cols);
    printf("Enter the filename: ");
    scanf("%s", filename);

    // Open the file for reading
    file = fopen(filename, "r");
    if (file == NULL) {
        printf("Error opening file.\n");
        return 1;
    }
    
    // Allocate memory for the matrix
    matrix = (int **)malloc(rows * sizeof(int *));
    for (int i = 0; i < rows; i++) {
        matrix[i] = (int *)malloc(cols * sizeof(int));
    }
    
    readFile(file, matrix, rows, cols);
    fclose(file);
    loadGreyScale(matrix, rows, cols);
    saveFilePPM(filename, matrix, rows, cols);

    // Free allocated memory
    for (int i = 0; i < rows; i++) {
        free(matrix[i]);
    }
    free(matrix);

    return 0;
}
```
# 평가
## 파일 생성
```py
with open("map-input-100-100.dat", "w") as f:
    for num in range(100, 0, -1):  # 100부터 1까지 반복
        f.write((str(num) + " ") * 100)  # 각 숫자를 100번 반복하여 파일에 작성

import random

with open("map-input-150-150.dat", "w") as f:
    for _ in range(22500):  # 랜덤 생성
        f.write(str(random.randint(1, 100)) + " ")

```
## 이미지 생성
[ppm 이미지를 png로](https://www.coolutils.com/online/PPM-to-PNG)

![Image](https://github.com/user-attachments/assets/06b095e2-0401-42ad-b0c7-696710500898)

![Image](https://github.com/user-attachments/assets/42163071-2bb5-44a3-80b4-894c2825645a)

# Appendix
## 2차원 배열의 구조
```c
int **matrix;

// Allocate memory for the matrix
matrix = (int **)malloc(rows * sizeof(int *));
for (int i = 0; i < rows; i++) {
    matrix[i] = (int *)malloc(cols * sizeof(int));
}

// Free allocated memory
for (int i = 0; i < rows; i++) {
    free(matrix[i]);
}
free(matrix);
```
rows는 3, cols는 4라고 하자

```c
matrix = (int **)malloc(rows * sizeof(int *));
```
를 통해 matrix에는 길이 3의 할당되었고, matrix에는 해당 배열의 시작 주소를 가지게 된다. 즉,

```
matrix <---0x64
----------------------------

--------------------------
????????|????????|???????? 메모리에 저장된 값
--------------------------
0x64     0x72     0x80   메모리 주소
```
와 같이 구성된다.
```c
for (int i = 0; i < rows; i++) {
    matrix[i] = (int *)malloc(cols * sizeof(int));
}
```
이제 길이 4를 가지는 배열 3개를 할당하고, 할당된 주소를 이전에 할당한 메모리 0x64 - 0x80에 저장한다.

```
matrix <---0x64
----------------------------

--------------------------------
0x00000100|0x00000228|0x00000356 메모리에 저장된 값
--------------------------------
0x64       0x72       0x80        메모리 주소


------------------------
?????|?????|?????|?????  메모리에 저장된 값
------------------------
0x100 0x132 0x164 0x196  메모리 주소

------------------------
?????|?????|?????|?????  메모리에 저장된 값
------------------------
0x228 0x260 0x292 0x324  메모리 주소

------------------------
?????|?????|?????|?????  메모리에 저장된 값
------------------------
0x356 0x388 0x420 0x452  메모리 주소


```
즉 matrix는 0x64 라는 값을 가진다.

matrix(1)은 $ *(matrix + (1\times sizeof(int*)))=*(0x64+8)=*(0x72) $ 로서 0x228의 값을 가진다.

matrix(1)(2)은 $ *(matrix[1] + (2\times sizeof(int)))=*(0x228+64)=*(0x292) $ 로서, 해당 배열에 저장된 값을 출력한다.