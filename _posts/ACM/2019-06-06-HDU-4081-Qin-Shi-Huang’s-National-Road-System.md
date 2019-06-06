---
layout: post
title:  "HDU 4081 Qin Shi Huang’s National Road System"
date:   2019-06-06
categories: ACM
---

[HDU 4081 Qin Shi Huang’s National Road System](http://acm.hdu.edu.cn/showproblem.php?pid=4081) 

## Problem Description
> **There were n cities in China and Qin Shi Huang wanted them all be connected by n-1 roads, in order that he could go to every city from the capital city Xianyang.**  
> Although Qin Shi Huang was a tyrant, he wanted the total length of all roads to be minimum, so that the road system may not cost too many people’s life. A daoshi (some kind of monk) named Xu Fu told Qin Shi Huang that he could build a road by magic and that magic road would cost no money and no labor. But Xu Fu could only build ONE magic road for Qin Shi Huang. So Qin Shi Huang had to decide where to build the magic road. Qin Shi Huang wanted the total length of all none magic roads to be as small as possible, but Xu Fu wanted the magic road to benefit as many people as possible —— **So Qin Shi Huang decided that the value of A/B (the ratio of A to B) must be the maximum, which A is the total population of the two cites connected by the magic road, and B is the total length of none magic roads.**  

## Input
> The first line contains an integer t meaning that there are t test cases(t <= 10).  
> For each test case:  
> The first line is an integer n meaning that there are n cities(2 < n <= 1000).  
> Then n lines follow. Each line contains three integers X, Y and P ( 0 <= X, Y <= 1000, 0 < P < 100000). (X, Y) is the coordinate of a city and P is the population of that city.  
> It is guaranteed that each city has a distinct location.  
```
2
4
1 1 20
1 2 30
200 2 80
200 1 100
3
1 1 20
1 2 30
2 2 40
```

## Output
> For each test case, print a line indicating the above mentioned maximum ratio A/B. The result should be rounded to 2 digits after decimal point.  
```
65.00
70.00
```

## 解析
1. 正常构建MST，但同时需要记录好构建出来的Tree，以及总长度cost
2. 通过BFS，遍历所有端点，记录maxd[i][j]，即在MST之中，i与j之间最长的边
3. 遍历每一对端点(i,j)，无论此时Edge(i,j)是否在MST之中，测试在此对端点间构建魔法边所得答案，即(w[i]+w[j])/(cost-maxd[i][j])，记录最大值即为最优解

## Source Code
```
//
//  main_.cpp
//  CodeForces
//
//  Created by 谷宇轩 on 6/6/2019.
//  Copyright © 2019 Yuxuan. All rights reserved.
//

#include <cstdio>
#include <cmath>
#include <cstring>
#include <queue>
#include <algorithm>
using namespace std;

const int MAXN = 1005;

struct point {
    int x, y, w;
} p[MAXN];

double dis(int i, int j) {
    return sqrt((p[i].x - p[j].x) * (p[i].x - p[j].x) + (p[i].y - p[j].y) * (p[i].y - p[j].y));
}

struct edge {
    int u, v;
    double w;
    bool operator < (const edge &rhs) const {
        return w < rhs.w;
    }
} e[MAXN * MAXN];

int treeEcnt, firstE[MAXN];
struct treeEdge {
    int to, nxt;
    double cost;
} treeE[MAXN << 1];

void addTreeE(int u, int v, double w) {
    treeE[treeEcnt].to = v;
    treeE[treeEcnt].nxt = firstE[u];
    treeE[treeEcnt].cost = w;
    firstE[u] = treeEcnt++;
}

int fa[MAXN];

int findfa (int u) {
    if (fa[u] < 0)
        return u;
    else
        return findfa(fa[u]);
}

void uninfa(int u, int v) {
    fa[u] = fa[u] + fa[v];
    fa[v] = u;
}

bool unin(int u, int v) {
    int fau = findfa(u), fav = findfa(v);
    if (fau == fav)
        return false;
    uninfa(fau, fav);
    return true;
}


double maxd[MAXN][MAXN];
bool vis[MAXN];

void bfs(int s) {
    queue<int> q;
    memset(vis, false, sizeof(vis));
    vis[s] = true;
    q.push(s);
    while (!q.empty()) {
        int u = q.front();
        q.pop();
        
        for (int i = firstE[u]; i != -1; i = treeE[i].nxt) {
            int v = treeE[i].to;
            if (vis[v])
                continue;
            maxd[s][v] = max(maxd[s][u], treeE[i].cost);
            maxd[v][s] = maxd[s][v];
            q.push(v);
            vis[v] = true;
        }
    }
}

int main() {
    int N, T;
    scanf("%d", &T);
    while (T--) {
        
        scanf("%d", &N);
        for (int i = 0; i < N; i++) {
            fa[i] = -1;
            scanf("%d%d%d", &p[i].x, &p[i].y, &p[i].w);
        }
        
        int edgeCnt = 0;
        for (int i = 0; i < N; i++) {
            for (int j = 0; j < N; j++) {
                e[edgeCnt].u = i;
                e[edgeCnt].v = j;
                e[edgeCnt++].w = dis(i, j);
            }
        }
        
        sort(e, e+edgeCnt);
        
        int leftP = N - 1;
        double cost = 0;
        
        treeEcnt = 0;
        memset(firstE, -1, sizeof(firstE));
        
        for (int i = 0; i < edgeCnt && leftP > 0; i++) {
            if (unin(e[i].u, e[i].v)) {
                leftP--;
                cost += e[i].w;
                addTreeE(e[i].u, e[i].v, e[i].w);
                addTreeE(e[i].v, e[i].u, e[i].w);
            }
        }
        
        memset(maxd, 0, sizeof(maxd));
        for (int i = 0; i < N; i++) {
            bfs(i);
        }
        
        double maxab = 0;
        for (int i = 0; i < N; i++) {
            for (int j = i + 1; j < N; j++) {
                maxab = max(maxab, (p[i].w + p[j].w) / (cost-maxd[i][j]));
            }
        }
        
        printf("%.2f\n", maxab);
        
    }
    
    return 0;
}
```
