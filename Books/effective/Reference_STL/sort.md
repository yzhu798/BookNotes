* std::sort整合了多种排序算法以尽可能达到最优效果：首先做条件判断，数据多用快速排序，数据少用插入排序，递归层次深用堆排序。下面是std::sort的源码
```cpp
template<class _RanIt, class _Pr>
inline
void sort(const _RanIt _First, const _RanIt _Last, _Pr _Pred)
{
    _Adl_verify_range(_First, _Last); // 判断迭代器范围
    const auto _UFirst = _Get_unwrapped(_First); // 相当于auto i = first
    const auto _ULast = _Get_unwrapped(_Last); // 相当于auto j = last
    _Sort_unchecked(_UFirst, _ULast, _ULast - _UFirst, _Pass_fn(_Pred)); // 真正的排序函数
}
```
* 跟进到排序函数
```cpp
template<class _RanIt, class _Pr>
inline
void _Sort_unchecked(_RanIt _First, _RanIt _Last, _Iter_diff_t<_RanIt> _Ideal, _Pr _Pred)
{
    _Iter_diff_t<_RanIt> _Count; // 迭代器的范围差
    // _ISORT_MAX是插入排序的最大尺寸，其实就是一个const int值，定义为32
    while (_ISORT_MAX < (_Count = _Last - _First) && 0 < _Ideal)
    {	// 数据量多于32，且传入的迭代器的范围差大于0，则用快速排序分治
        auto _Mid = _Partition_by_median_guess_unchecked(_First, _Last, _Pred); // 分割算法
        _Ideal = (_Ideal >> 1) + (_Ideal >> 2);	// 每次乘0.75，即分割1.5*log2(N)次

        // 迭代数据量多的部分
        if (_Mid.first - _First < _Last - _Mid.second)
        {
            _Sort_unchecked(_First, _Mid.first, _Ideal, _Pred);
            _First = _Mid.second;
        }
        else
        {
            _Sort_unchecked(_Mid.second, _Last, _Ideal, _Pred);
            _Last = _Mid.first;
        }
    }

    if (_ISORT_MAX < _Count)
    {    // 分割次数过多则使用堆排序
        _Make_heap_unchecked(_First, _Last, _Pred);
        _Sort_heap_unchecked(_First, _Last, _Pred);
    }
    else if (2 <= _Count)
    {    // 如果数据量小于32但不少于2，则使用插入排序
        _Insertion_sort_unchecked(_First, _Last, _Pred);
    }
}
```
