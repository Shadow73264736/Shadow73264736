import threading
import multiprocessing
import win32process

WORKER_COUNT = 16
THREAD_COUNT_PER_WORKER = 50

def thread_func(worker_num, thread_num, thread_barrier, thread_event):
    thread_barrier.wait()
    thread_event.wait()
    print(f"Hello from worker {worker_num}, thread {thread_num}")

def worker_func(worker_num, worker_barrier):
    cpu_num = worker_num % multiprocessing.cpu_count()
    win32process.SetProcessAffinityMask(-1, 1 << cpu_num)

    thread_barrier = threading.Barrier(THREAD_COUNT_PER_WORKER + 1)
    thread_event = threading.Event()
    threads = [
        threading.Thread(
            target=thread_func,
            args=(worker_num, thread_num, thread_barrier, thread_event)
        )
        for thread_num in range(THREAD_COUNT_PER_WORKER)
    ]

    for thread in threads:
        thread.start()
    thread_barrier.wait()
    worker_barrier.wait()
    thread_event.set()
    for thread in threads:
        thread.join()
    
if __name__ == "__main__":
    worker_barrier = multiprocessing.Barrier(WORKER_COUNT + 1)
    workers = [
        multiprocessing.Process(
            target=worker_func,
            args=(worker_num, worker_barrier)
        )
        for worker_num in range(WORKER_COUNT)
    ]
    for worker in workers:
        worker.start()
    worker_barrier.wait()
    for worker in workers:
        worker.join()
