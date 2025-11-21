// net_stress_hp.cpp
#include <chrono>
#include <curl/curl.h>
#include <iostream>
#include <string>
#include <thread>
#include <vector>

#if defined(_WIN32)
#include <windows.h>
#else
#include <pthread.h>
#include <sys/resource.h>
#include <unistd.h>
#endif

static size_t sink(char* ptr, size_t size, size_t nmemb, void*) {
    return size * nmemb; // 丢弃数据
}

void set_process_high_priority() {
#if defined(_WIN32)
    SetPriorityClass(GetCurrentProcess(), HIGH_PRIORITY_CLASS);
#else
    // nice 值越低优先级越高，这里尝试 -10；需要足够权限
    setpriority(PRIO_PROCESS, 0, -10);
#endif
}

void set_thread_high_priority() {
#if defined(_WIN32)
    SetThreadPriority(GetCurrentThread(), THREAD_PRIORITY_HIGHEST);
#else
    sched_param sch_params;
    sch_params.sched_priority = sched_get_priority_max(SCHED_FIFO);
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &sch_params); // 可能需要 root
#endif
}

void worker(const std::string& url, int id, int delay_ms) {
    set_thread_high_priority();

    CURL* curl = curl_easy_init();
    if (!curl) {
        std::cerr << "Worker " << id << " failed to init curl\n";
        return;
    }
    curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, sink);
    curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L);
    // 如目标支持持久连接，可启用以下选项以减少握手开销：
    curl_easy_setopt(curl, CURLOPT_TCP_KEEPALIVE, 1L);

    while (true) {
        CURLcode res = curl_easy_perform(curl);
        if (res != CURLE_OK) {
            std::cerr << "Worker " << id << " error: " << curl_easy_strerror(res) << "\n";
        }
        if (delay_ms > 0) {
            std::this_thread::sleep_for(std::chrono::milliseconds(delay_ms));
        }
    }
    curl_easy_cleanup(curl);
}

int main(int argc, char* argv[]) {
    if (argc < 5) {
        std::cerr << "Usage: " << argv[0] << " <url> <num_threads> <delay_ms> <runtime_seconds>\n";
        std::cerr << "Example: " << argv[0]
                  << " http://localhost:8000/largefile 32 0 120\n";
        return 1;
    }
    std::string url = argv[1];
    int num_threads = std::stoi(argv[2]);
    int delay_ms = std::stoi(argv[3]);      // 可设为 0
    int runtime_seconds = std::stoi(argv[4]);

    if (num_threads <= 0 || delay_ms < 0 || runtime_seconds <= 0) {
        std::cerr << "Invalid arguments.\n";
        return 1;
    }

    set_process_high_priority();

    curl_global_init(CURL_GLOBAL_DEFAULT);
    std::vector<std::thread> threads;
    threads.reserve(num_threads);
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back(worker, url, i, delay_ms);
    }

    std::this_thread::sleep_for(std::chrono::seconds(runtime_seconds));
    std::cerr << "Stopping after " << runtime_seconds << "s\n";
    std::quick_exit(0);
}
