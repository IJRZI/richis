# priority queue with updatable priority

事情是这样的，我正在维护一组会话，每个会话每隔一段时间都需要被轮询，每个会话轮询的间隔都是不等且变化的 —— 每次轮询之后，它会告诉我下次什么时候来重新轮询

当然，在那个它告诉我的 “下次轮询时间” 之前轮询它并不会有什么作用（也没有副作用），在那个时间之后轮询则会产生一点效率损失，比如会话上的包被更晚处理了，之类的

因此一开始我的实现非常粗暴：持续地以一个非常小的时间间隔 tick，每个 tick 都遍历所有的会话并轮询它们。但假设一个会话 100ms 后才需要再次轮询，我却每 5ms 都轮询一遍所有会话，显然做了许多无用功

于是我希望用优先队列来安排每个会话的轮询时间，当然，最先想到的就是 `std::priority_queue`

但还有一些需求无法被满足，因为会话的轮询还有一个变量：**会话的下次轮询时间可能会改变**
- 假如会话 A 原本计于 100ms 后被再次轮询，但其上产生了一个新的消息（或是什么事件），则会话 A 可能需要在 10ms 后就被轮询

放到优先队列的语境里，即一个队列内元素的优先级可能会被改变，但 `std::priority_queue` 并不提供改变优先级的接口，事实上它也不能做到，因为它基于堆实现，想修改堆内任意一个元素的值而不破坏堆的性质，需要以 O(N) 的代价重建堆来实现，但我们期望一个 O(log n) 的更新手段

类似的需求也在 Dijkstra 算法中被用到，其解决办法在 [Easiest way of using min priority queue with key update in C++](https://stackoverflow.com/a/27305600) 中被提到：在 Dijkstra 算法对优先队列的需求中，可以通过 "lazy deletion" 来实现类似的功能，即不考虑更新元素的优先级，而是直接将更小优先级的元素插入队列中，新插入的元素总是会比旧元素更早被 `pop`，只需要加上去重逻辑，就变相 “更新” 了更新旧元素的优先级

但是在本项目中，我觉得这个方法并不适用，因为在 Dijkstra 中图的大小是 bounded 的，而会话上消息的产生是 unbounded 的，如果每次有新消息产生并要更新会话优先级时，都插入一个新元素，那 priority queue 可能会爆掉，或者产生效率问题（比如 `pop` 的时候需要去除大量重复元素）

因此，我希望有这样一个优先队列，它能够以某个优先级安排队列内所有元素的顺序，又可以快速地更新某个元素的优先级

## 实现
一个简单的实现如下
```c++
template <typename Priority, typename Val>
class UniquePriorityQueue
{
public:
    void push_or_update(const Priority &p, const Val &v)
    {
        auto vp_it = _vp_map.find(v); // O(log n)
        if (vp_it != _vp_map.end())
        { // update
            auto pv_it = _pv_map.find(vp_it->second); // O(log n)
            assert(pv_it != _pv_map.end());
            _pv_map.erase(pv_it); // O(1)

            vp_it->second = p;
            _pv_map.insert({vp_it->second, vp_it->first}); // O(log n)
        }
        else
        {
            auto [vp_it, success] = _vp_map.emplace(v, p); // O(log n)
            assert(success);
            _pv_map.insert({vp_it->second, vp_it->first}); // O(log n)
        }
    }

    void pop()
    {
        auto pv_it = _pv_map.begin();
        assert(pv_it != _pv_map.end());
        auto vp_it = _vp_map.find(pv_it->second);
        assert(vp_it != _vp_map.end());
        _pv_map.erase(pv_it);
        _vp_map.erase(vp_it);
    }

    const std::pair<const Priority&, const Val&> top() const {
        auto pv_it = _pv_map.begin();
        assert(pv_it != _pv_map.end());
        return {pv_it->first, pv_it->second};
    }

    bool empty() const {
        assert(_pv_map.size() == _vp_map.size());
        return _vp_map.empty();
    }
private:
    std::map<Priority, const Val &> _pv_map;
    std::unordered_map<Val, Priority> _vp_map;
};
```
之所以叫 `UniquePriorityQueue`，是因为队列内的 `Val` 是不重复的，如果一个 `Val` 已经存在，那么 `push_or_update` 的语义就是更新其优先级

各接口的复杂度为：
- O(1): `top()`, `empty()`
- O(log n): `push_or_update()`, `pop()`

以下是一个完整可编译的程序(C++20), 模拟了刚才所说的安排会话的例子
```c++
#include <map>
#include <unordered_map>
#include <vector>
#include <memory>
#include <cassert>
#include <random>

template <typename Priority, typename Val>
class UniquePriorityQueue
{
public:
    void push_or_update(const Priority &p, const Val &v)
    {
        auto vp_it = _vp_map.find(v);
        if (vp_it != _vp_map.end())
        { // update
            auto pv_it = _pv_map.find(vp_it->second);
            assert(pv_it != _pv_map.end());
            _pv_map.erase(pv_it);

            vp_it->second = p;
            _pv_map.insert({vp_it->second, vp_it->first});
        }
        else
        {
            auto [vp_it, success] = _vp_map.emplace(v, p);
            assert(success);
            _pv_map.insert({vp_it->second, vp_it->first});
        }
    }

    void pop()
    {
        auto pv_it = _pv_map.begin();
        assert(pv_it != _pv_map.end());
        auto vp_it = _vp_map.find(pv_it->second);
        assert(vp_it != _vp_map.end());
        _pv_map.erase(pv_it);
        _vp_map.erase(vp_it);
    }

    const std::pair<const Priority&, const Val&> top() const {
        auto pv_it = _pv_map.begin();
        assert(pv_it != _pv_map.end());
        return {pv_it->first, pv_it->second};
    }

    bool empty() const {
        assert(_pv_map.size() == _vp_map.size());
        return _vp_map.empty();
    }
private:
    std::map<Priority, const Val &> _pv_map;
    std::unordered_map<Val, Priority> _vp_map;
};

struct Job
{
    int id;
    int i = 0;
    std::vector<int> schedule;

    Job(int id) : id(id) {}

    int next()
    {
        if (i < schedule.size())
        {
            return schedule[i++];
        }
        else
        {
            return INT32_MAX;
        }
    }
};

constexpr int JOBS = 1000;
constexpr int ITERATIONS = 1000000;

int main()
{
    std::srand(time(nullptr));

    UniquePriorityQueue<int, std::shared_ptr<Job>> q;

    std::vector<std::shared_ptr<Job>> jobs;
    std::vector<Job *> schedule;

    // generate N jobs
    for (int i = 0; i < JOBS; i++)
    {
        jobs.push_back(std::make_shared<Job>(i));
    }

    // tick ITERATIONS times, each tick one random job should be done, use `schedule` to record the order
    for (int priority = 0; priority < ITERATIONS; priority++)
    {
        auto job = jobs[std::rand() % JOBS];
        schedule.push_back(job.get());

        job->schedule.push_back(priority);
    }

    // init priority queue
    for (auto& job: jobs)
    {
        auto next = job->next();
        q.push_or_update(next, job);
    }

    // tick, each tick get the job with the lowest priority, and update its priority
    for (int i = 0; i < ITERATIONS; i++)
    {
        printf("\rticking %d/%d", i, ITERATIONS);
        const auto &[p, job] = q.top();

        // check if the order is correct
        assert(p == i);
        assert(job.get() == schedule[i]);

        // schedule next
        auto next = job->next();
        assert(next > p);

        q.push_or_update(next, job);
    }
    assert(q.top().first == INT32_MAX);
    return 0;
}
```
