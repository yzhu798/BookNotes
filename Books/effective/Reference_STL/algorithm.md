> \*标注为`C++11/14`引入，范围为`[first,last)`

## Non-modifying sequence operations
|函数名|功能|形式|f|
|:-:|:-:|:-:|:-:|
|*all_of|检查范围内是否**所有**元素满足给定条件，返回一个`bool`|`all_of`(v.begin(), v.end(), f)|bool f(int i);f也可以是一个`lambda`|
|*any_of|检查范围内是否**存在**元素满足给定条件，返回一个`bool`|`any_of`(v.begin(), v.end(), f)|bool f(int i);f也可以是一个`lambda`|
|*none_of|检查范围内是否**所有元素不满足**给定条件，返回一个`bool`|`none_of`(v.begin(), v.end(), f)|bool f(int i);f也可以是一个`lambda`|
|for_each|**遍历范围**内元素，传给f执行|`for_each`(v.begin(), v.end(), f)|void f(int i);f也可以是一个只有void `operator()`(int i){}的**类对象**|
|find|查找范围内是否存在30，若存在则返回第一个对应**迭代器**，否则返回尾迭代器|`find`(v.begin(), v.end(), 30)||
|find_if|查找范围内是否存在满足给定条件的元素，若存在则返回第一个对应迭代器，否则返回尾迭代器|`find_if`(v.begin(), v.end(), f)|bool f(int i)|
|*find_if_not|查找范围内是否存在不满足给定条件的元素，若存在则返回第一个对应**迭代器**，否则返回尾迭代器|`find_if_not`(v.begin(), v.end(), f)|bool f(int i)|
|find_end|范围2为范围1的最后的子序列，返回范围2首元素在范围1中的**对应的迭代器**，若未找到则返回范围1的**尾迭代器**，参数可以多一个自定义条件f|`find_end`(v.begin(), v.end(), sub.begin(), sub.end(), f)|bool f(int i, int j)|
|find_first_of|范围2为范围1的第一个子序列，返回范围2首元素在范围1中的对应的迭代器，若未找到则返回范围1的尾迭代器，参数可以多一个自定义条件f|`find_first_of`(v.begin(), v.end(), sub.begin(), sub.end(), f)|bool f(int i, int j)|
|adjacent_find|**查找第一对相邻且相同的元素**，返回后一个元素的迭代器，若未找到则返回尾迭代器，参数可以多一个自定义条件f|`adjacent_find`(v.begin(), v.end(), f)|bool f(int i, int j)|
|count|返回值为范围内30出现的次数|`count`(v.begin(), v.end(), 30)||
|count_if|返回范围内满足**给定条件**的元素出现的次数|`count_if`(v.begin(), v.end(), f)|bool f(int i)|
|mismatch|返回范围内**不匹配**给定数组的两个不同的值组成的`pair`，参数可以多一个自定义条件f。如果都相同则内存溢出|`mismatch`(v.begin(), v.end(), arr, f)|bool f(int i, int j)|
|equal|查找范围内的元素是否与**数组**完全相同，**返回一个bool**，参数可以多一个自定义条件f。如果元素数量不同则内存溢出|`equal`(v.begin(), v.end(), arr, f)|bool f(int i, int j)|
|*is_permutation|查找范围内是否含有相同的元素，**顺序和数量可以不同**|`is_permutation`(v.begin(), v.end(), arr)||
|search|范围2为范围1的第一个子序列，返回范围2首元素在范围1中的**对应的迭代器**，若未找到则返回范围1的尾迭代器，参数可以多一个自定义条件f|`search`(v.begin(), v.end(), sub.begin(), sub.end(), f)|bool f(int i, int j)|
|search_n|查找范围内出现**2个连续**的30的的**第一个**30对应的迭代器，未找到则返回尾迭代器，参数可以多一个自定义条件f|`search_n`(v.begin(), v.end(), 2, 30, f)|bool f(int i, int j)|

## Modifying sequence operations
|函数名|功能|形式|f|
|:-:|:-:|:-:|:-:|
|copy|把范围1的内容拷贝到设置好size的v中|copy(arr, arr+7, v.begin())||
|*copy_n|把arr的7个元素拷贝到设置好size的v中|copy_n(arr, 7, v.begin())||
|*copy_if|把满足条件的v中元素拷贝到b中，返回目标范围的尾迭代器|copy_if (v.begin(), v.end(), b.begin(), f)|bool f(int i)|
|copy_backforward|把范围内元素逆序从后往前拷贝|copy_backward(v.begin(), v.begin()+5, v.end())||
|*move|把范围1的元素移动到bar中，或直接move整个容器|move(v.begin(), v.begin()+4, bar.begin());foo = std::move(bar)||
|*move_backforward|把范围1的元素逆序从后往前移动|move_backward (elems,elems+4,elems+5)||
|swap|交换元素或容器的全部内容|swap(x, y);swap(foo, bar)||
|swap_ranges|将范围1内的元素和bar对应数量的元素交换|swap_ranges(foo.begin()+1, foo.end()-1, bar.begin())||
|iter_swap|交换两个迭代器的元素|iter_swap(arr+1,v.begin()+2)||
|transform|对foo所有元素进行op_increase操作，拷贝到bar|transform (foo.begin(), foo.end(), bar.begin(),  op_increase);transform (foo.begin(), foo.end(), bar.begin(), foo.begin(), std::plus<int>())||
|replace|把范围内的20换成99|replace(v.begin(), v.end(), 20, 99)||
|replace_if|把范围内满足条件的元素换成0|replace_if(v.begin(), v.end(), f, 0)|bool f(int i)|
|replace_copy|把范围中的20以99拷贝到v（范围元素不变）|replace_copy(arr, arr+8, v.begin(), 20, 99)||
|replace_copy_if|把范围内满足条件的元素以0拷贝到bar|replace_copy_if (foo.begin(), foo.end(), bar.begin(), f, 0)|bool f(int i)|
|fill|把范围内元素换成5|fill(v.begin(),v.begin()+4,5)||
|fill_n|把4个20拷贝到对应位置|fill_n(v.begin()+3,4,20)||
|generate|将f的执行结果拷贝到范围内|generate(v.begin(), v.end(), f)|int f() { return (std::rand()%100); }|
|generate_n|将f的执行结果拷贝到指定位置开始的n个元素中|generate_n (arr, 9, f)|int f()|
|remove|将范围内的20删除|remove(v.begin(), v.end(), 20)||
|remove_if|将范围内满足条件的元素删除|remove_if (v.begin(), v.end(), f)|bool f(int i)|
|remove_copy|将范围内的20删除拷贝到v（范围元素不变）|remove_copy (arr,arr+8,v.begin(),20)||
|remove_copy_if|将范围内满足条件的元素删除拷贝到v|remove_copy_if (arr,arr+9,v.begin(),f)|bool f(int i)|
|unique|去除重复元素，也可以自定义判断条件|unique(v.begin(), v.end());unique(v.begin(), v.end(),f)|默认为bool f(int i, int j) { return (i==j); }|
|unique_copy|去除范围内的重复元素拷贝到v，可以自定义判断条件|unique_copy(arr,arr+9,v.begin());unique_copy(arr,arr+9,v.begin(),f)|默认为bool f(int i, int j) { return (i==j); }|
|reverse|反转范围内元素|reverse(v.begin(),v.end())||
|reverse_copy|将反转的范围内元素拷贝到v|reverse_copy(arr, arr+9, v.begin())||
|rotate|交换[first,middle)和[middle,last)两个范围内元素|rotate(v.begin(),v.begin()+3,v.end())||
|rotate_copy|将交换的结果拷贝到v|rotate_copy(arr,arr+3,arr+7,v.begin())||
|random_shuffle|随机打乱范围内元素位置|random_shuffle(v.begin(),v.end());random_shuffle(v.begin(),v.end(), f)|int f(int i) { return std::rand()%i;}|
|*shuffle|用随机数引擎打乱范围内元素位置|unsigned seed = std::chrono::system_clock::now().time_since_epoch().count();shuffle(v.begin(), v.end(), std::default_random_engine(seed));||

## Partitions
|函数名|功能|形式|f|
|:-:|:-:|:-:|:-:|
|*is_partitioned|检测范围**是否按条件划分**，返回一个bool|is_partitioned(v.begin(),v.end(),f)|bool f(int i)|
|partition|按条件划分范围，返回划分**边界的迭代器**，不稳定|partition(v.begin(), v.end(), f)|bool f(int i)|
|stable_partition|按条件划分，返回划分**边界的迭代器**，稳定|stable_partition(v.begin(), v.end(), f)|bool f(int i)|
|*partition_copy|范围内满足条件的**划分**到两个**新容器内**|partition_copy(v.begin(), v.end(), odd.begin(), even.begin(), IsOdd)||
|*partition_point|返回第一个**不满足条件的迭代器**，不存在则返回尾迭代器|partition_point(v.begin(),v.end(),IsOdd)||

## Sorting
|函数名|功能|形式|f|
|:-:|:-:|:-:|:-:|
|sort|排序，默认用<排序，可以自定义条件|sort(v.begin(), v.end());sort(v.begin(), v.end(), f)|bool f(int i,int j) { return (i<j); };struct myclass { bool operator()(int i,int j) { return (i<j);} } f;|
|stable_sort|稳定排序|同上|bool f(double i,double j){ return (int(i)<int(j)); }|
|partial_sort|前5个最小的元素从小到大排序，剩下的未排序|partial_sort(v.begin(), v.begin()+5, v.end());partial_sort(v.begin(), v.begin()+5, v.end(), f)|bool f(int i,int j) { return (i<j); }|
|partial_sort_copy|把部分排序的结果拷贝到v|partial_sort_copy(arr, arr+9, v.begin(), v.end());partial_sort_copy(arr, arr+9, v.begin(), v.end(), f)|bool f(int i,int j) { return (i<j); }|
|*is_sorted|返回一个`bool`判断是否已排序|std::is_sorted(v.begin(),v.end())||
|*is_sorted_until|返回第一个未排序的迭代器，若不存在则返回尾迭代器|is_sorted_until(v.begin(),v.end())||
|nth_element|将第5大的元素放到第5的位置，比它小的在前，大的在后|nth_element(v.begin(), v.begin()+5, v.end());nth_element(v.begin(), v.begin()+5, v.end(), f)|bool f(int i,int j) { return (i<j); }|

## Binary search
|函数名|功能|形式|f|
|:-:|:-:|:-:|:-:|
|lower_bound|返回第一个大于等于20的元素的迭代器|lower_bound (v.begin(), v.end(), 20)||
|upper_bound|返回第一个大于20的元素的迭代器|upper_bound (v.begin(), v.end(), 20)||
|equal_range|返回一个part，part.first为lower_bound，part.second为upper_bound，如果f为>则相反|equal_range (v.begin(), v.end(), 20);equal_range (v.begin(), v.end(), 20, f)|bool f(int i,int j) { return (i>j); }|
|binary_search|二分查找，返回一个bool|binary_search (v.begin(), v.end(), 3);binary_search (v.begin(), v.end(), 3, f)|bool f(int i,int j) { return (i<j); }|

## Merge
|函数名|功能|形式|f|
|:-:|:-:|:-:|:-:|
|merge|将两个有序序列合并到v并排序|merge (first,first+5,second,second+5,v.begin())||
|inplace_merge|连接在一起的序列[first,middle)和[middle,last)都已排序，将其归并为单一有序序列|inplace_merge(v.begin(),v.begin()+5,v.end())||
|includes|返回一个bool，判断arr2是否为arr1的子集|includes(arr1,arr1+10,arr2,arr2+4);includes(arr1,arr1+10,arr2,arr2+4,f)|bool f(int i,int j) { return (i<j); }|
|set_union|将两个集合的并集拷贝到v，返回并集元素数后一个迭代器|set_union(first, first+5, second, second+5, v.begin())||
|set_intersection|交集|同上||
|set_difference|范围1减去范围2的差集|同上||
|set_symmetric_difference|对称差|同上||

## Heap
|函数名|功能|形式|
|:-:|:-:|:-:|
|push_heap|向堆中加一个元素| v.push_back(99);push_heap(v.begin(),v.end())|
|pop_heap|删除堆中最大元素|pop_heap(v.begin(),v.end())|
|make_heap|用指定范围造一个堆|make_heap(v.begin(),v.end())|
|sort_heap|堆排序|sort_heap(v.begin(),v.end())|
|*is_heap|判断是否为二叉堆结构，返回一个bool|is_heap(v.begin(),v.end())|
|*is_heap_until|返回第一个破坏二叉堆结构的迭代器|is_heap_until(v.begin(),v.end())|

## Min/max
|函数名|功能|形式|f|
|:-:|:-:|:-:|:-:|
|min|返回较小元素|min('a','z')||
|max|返回较大元素|max(3.3, 2.5)||
|*minmax|返回一个`pair`，pair.first为最小值，pair.second为最大值|auto result = minmax({1,2,3,4,5})||
|min_element|返回范围内**最小值的迭代器**|min_element(myints,myints+7);，min_element(myints,myints+7,f)|bool myfn(int i, int j) { return i<j; };struct myclass { bool operator() (int i,int j) { return i<j; }} f;|
|max_element|返回范围内最大值的迭代器|同上||
|*minmax_element|返回一个`pair`，pair.first为最小值的迭代器，pair.second为最大值的**迭代器**|auto result = std::minmax_element(v.begin(),v.end())||

## Other
|函数名|功能|形式|
|:-:|:-:|:-:|
|lexicographical_compare|按**字典序比较**两个序列|lexicographical_compare(foo,foo+5,bar,bar+9, compare)|
|next_permutation|范围按字典序的**下一个排列组合**，若没有则返回false|next_permutation(arr,arr+3)|
|prev_permutation|范围按字典序的**上一个排列组合**，若没有则返回false|prev_permutation(arr,arr+3)|
