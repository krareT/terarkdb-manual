- **OPS** 表示每秒**插入**或**搜索**多少个 Key（1 M op/sec 表示每秒 100万 次操作）
- **吞吐率**表示每秒**插入**或**搜索**的 Key 的总长度
- **顺序插入** 或 **顺序搜索** 表示按照 Key 的 Bytewise 顺序从小到大执行插入或搜索
- **倍率** 指 rbtree 或 sort 的性能是 Patricia Trie 的百分之多少
- **顺序 LB** 和 **随机 LB** 中的 **LB** 指 `lower_bound`

作为对比，我们加入了 sort 的对比，相当于对一个 `StringVector` 的操作
- `StringVector` 进行了深度优化：
  - 使用一块连续内存保存所有的 StringData
  - 另外一块连续内存是索引，指向 StringData 中的偏移和相应 String 的长度
- sort 列的**插入**指在 `StringVector` 上逐个 Append String，然后进行排序
  - Append 开始前预分配大小刚够容纳所有 String 的内存
- sort 列的**搜索**指在 `StringVector` 上使用 std::lower_bound 语义执行搜索

<table>
<tr>
  <th rowspan="3" valign="top">
     FileList<br/>作为<br/>测试数据
  </th>
  <td align="left" colspan="8">
    avglen = 76.362 &nbsp;&nbsp;&nbsp; lines = 1,307,768 &nbsp;&nbsp;&nbsp; size = 101,171,201
  </td>
</tr>
<tr>
  <th colspan="3">RbTree</th>
  <th colspan="3">sort</th>
  <th colspan="2">PatriciaTrie</th>
</tr>
<tr>
  <th>OPS<br/>M/s</th>
  <th>吞吐率<br/>MB/s</th>
  <th>倍率<br/>%</th>
  <th>OPS<br/>M/sec</th>
  <th>吞吐率<br/>MB/s</th>
  <th>倍率<br/>%</th>
  <th>OPS<br/>M/sec</th>
  <th>吞吐率<br/>MB/s</th>
</tr>
<tr>
  <td>内存占用</td>
  <th align="center" colspan="2">194.023 MB</th>
  <td align="right">483.45%</td>
  <th align="center" colspan="2">110.326 MB</th>
  <td align="right">274.90%</td>
  <th align="center" colspan="2">40.133 MB</th>
</tr>
<tr>
  <td>顺序插入</td>
  <td align="right">1.808</td>
  <td align="right">138.06</td>
  <td align="right">57.96%</td>
  <td align="right">2.423</td>
  <td align="right">185.06</td>
  <td align="right">77.69%</td>
  <td align="right">2.574</td>
  <td align="right">238.22</td>
</tr>
<tr>
  <td>随机插入</td>
  <td align="right">0.632</td>
  <td align="right">48.25</td>
  <td align="right">38.32%</td>
  <td align="right">1.267</td>
  <td align="right">96.75</td>  
  <td align="right">76.84%</td>  
  <td align="right">1.649</td>
  <td align="right">125.91</td>
</tr>
<tr>
  <td>顺序点查</td>
  <td align="right">2.943</td>
  <td align="right">224.70</td>
  <td align="right">51.08%</td>
  <td align="right">2.527</td>
  <td align="right">192.94</td>
  <td align="right">43.85%</td>
  <td align="right">5.761</td>
  <td align="right">439.93</td>
</tr>
<tr>
  <td>随机点查</td>
  <td align="right">0.626</td>
  <td align="right">47.82</td>
  <td align="right">33.90%</td>
  <td align="right">1.251</td>
  <td align="right">95.55</td>
  <td align="right">36.91%</td>
  <td align="right">1.769</td>
  <td align="right">135.11</td>
</tr>
<tr>
  <td>顺序 LB</td>
  <td align="right">2.943</td>
  <td align="right">224.70</td>
  <td align="right">51.08%</td>
  <td align="right">2.527</td>
  <td align="right">192.94</td>
  <td align="right">43.85%</td>
  <td align="right">5.761</td>
  <td align="right">439.93</td>
</tr>
<tr>
  <td>随机 LB</td>
  <td align="right">0.626</td>
  <td align="right">47.82</td>
  <td align="right">33.90%</td>
  <td align="right">1.251</td>
  <td align="right">95.55</td>
  <td align="right">36.91%</td>
  <td align="right">1.769</td>
  <td align="right">135.11</td>
</tr>
</table>

<table>
<tr>
  <th rowspan="3" valign="top">
     RandKey<br/>作为<br/>测试数据
  </th>
  <td align="left" colspan="8">
    avglen = 5.982 &nbsp;&nbsp;&nbsp; lines = 865,910 &nbsp;&nbsp;&nbsp; size = 6,045,903
  </td>
</tr>
<tr>
  <th colspan="3">RbTree</th>
  <th colspan="3">sort</th>
  <th colspan="2">PatriciaTrie</th>
</tr>
<tr>
  <th>OPS<br/>M/s</th>
  <th>吞吐率<br/>MB/s</th>
  <th>倍率<br/>%</th>
  <th>OPS<br/>M/sec</th>
  <th>吞吐率<br/>MB/s</th>
  <th>倍率<br/>%</th>
  <th>OPS<br/>M/sec</th>
  <th>吞吐率<br/>MB/s</th>
</tr>
<tr>
  <td>内存占用</td>
  <th align="center" colspan="2">67.526 MB</th>
  <td align="right">334.90%</td>
  <th align="center" colspan="2">12.973 MB</th>
  <td align="right">53.04%</td>
  <th align="center" colspan="2">24.457 MB</th>
</tr>
<tr>
  <td>顺序插入</td>
  <td align="right">2.36</td>
  <td align="right">14.10</td>
  <td align="right">42.11%</td>
  <td align="right">6.89</td>
  <td align="right">41.24</td>
  <td align="right">123.17%</td>
  <td align="right">5.60</td>
  <td align="right">33.49</td>
</tr>
<tr>
  <td>随机插入</td>
  <td align="right">1.29</td>
  <td align="right">7.74</td>
  <td align="right">31.00%</td>
  <td align="right">3.49</td>
  <td align="right">20.89</td>  
  <td align="right">83.73%</td>  
  <td align="right">4.17</td>
  <td align="right">24.95</td>
</tr>
<tr>
  <td>顺序点查</td>
  <td align="right">5.43</td>
  <td align="right">32.46</td>
  <td align="right">21.23%</td>
  <td align="right">6.42</td>
  <td align="right">38.38</td>
  <td align="right">25.09%</td>
  <td align="right">25.57</td>
  <td align="right">152.94</td>
</tr>
<tr>
  <td>随机点查</td>
  <td align="right">1.27</td>
  <td align="right">7.60</td>
  <td align="right">10.53%</td>
  <td align="right">2.14</td>
  <td align="right">12.79</td>
  <td align="right">17.72%</td>
  <td align="right">12.07</td>
  <td align="right">72.20</td>
</tr>
<tr>
  <td>顺序 LB</td>
  <td align="right">5.72</td>
  <td align="right">34.24</td>
  <td align="right">34.02%</td>
  <td align="right">6.69</td>
  <td align="right">40.00</td>
  <td align="right">39.75%</td>
  <td align="right">16.82</td>
  <td align="right">100.64</td>
</tr>
<tr>
  <td>随机 LB</td>
  <td align="right">1.30</td>
  <td align="right">7.76</td>
  <td align="right">14.10%</td>
  <td align="right">2.153</td>
  <td align="right">12.88</td>
  <td align="right">23.39%</td>
  <td align="right">9.21</td>
  <td align="right">55.06</td>
</tr>
</table>

- shuf 列的顺序插入指生成 `vector<size_t>`，shuf 列的随机插入指按照 shuf 序填充 `StringVector`：从源中随机取，往目标中顺序填


1. 插入性能大约是 rbtree 的 200%
2. 精确搜索的性能至少要达到 rbtree 的 300%
3. 范围搜索 (lower_bound) 至少要与 rbtree 持平
4. 最坏情况下的内存占用不能超过 rbtree
