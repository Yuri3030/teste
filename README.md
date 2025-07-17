#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <time.h>

#define PRODUCER_NUM 3
#define CONSUMER_NUM 3
#define BUFFER_SIZE 10

typedef struct Clock {
    int p[3];
} Clock;

Clock taskQueue[BUFFER_SIZE];
int taskCount = 0;

pthread_mutex_t mutex;
pthread_cond_t condFull;
pthread_cond_t condEmpty;

void submitClock(Clock clock) {
    pthread_mutex_lock(&mutex);

    while (taskCount == BUFFER_SIZE) {
        pthread_cond_wait(&condFull, &mutex);
    }

    taskQueue[taskCount] = clock;
    taskCount++;

    if (taskCount == BUFFER_SIZE) {
        printf(">> [Fila Cheia] Nenhum espaço disponível. Produtores aguardando...\n");
    }

    pthread_mutex_unlock(&mutex);
    pthread_cond_signal(&condEmpty);
}

Clock getClock() {
    pthread_mutex_lock(&mutex);

    while (taskCount == 0) {
        pthread_cond_wait(&condEmpty, &mutex);
    }

    Clock clock = taskQueue[0];
    for (int i = 0; i < taskCount - 1; i++) {
        taskQueue[i] = taskQueue[i + 1];
    }
    taskCount--;

    if (taskCount == 0) {
        printf("<< [Fila Vazia] Nenhum item disponível. Consumidores aguardando...\n");
    }

    pthread_mutex_unlock(&mutex);
    pthread_cond_signal(&condFull);

    return clock;
}

void *producerThread(void* arg) {
    long id = (long)arg;
    while (1) {
        Clock clock;
        for (int i = 0; i < 3; i++) {
            clock.p[i] = rand() % 100;  // clock entre 0 e 99
        }
        printf("[Produtor %ld] Gerou clock: (%d, %d, %d)\n", id, clock.p[0], clock.p[1], clock.p[2]);
        submitClock(clock);
        sleep(1);
    }
    return NULL;
}

void *consumerThread(void* arg) {
    long id = (long)arg;
    while (1) {
        Clock clock = getClock();
        clock.p[id]++;
        printf("[Consumidor %ld] Executou tarefa, Clock: (%d, %d, %d)\n", id, clock.p[0], clock.p[1], clock.p[2]);
        sleep(1);
    }
    return NULL;
}

int main() {
    srand(time(NULL));

    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&condEmpty, NULL);
    pthread_cond_init(&condFull, NULL);

    pthread_t producers[PRODUCER_NUM];
    pthread_t consumers[CONSUMER_NUM];

    for (long i = 0; i < PRODUCER_NUM; i++) {
        pthread_create(&producers[i], NULL, producerThread, (void*)i);
        sleep(1);
    }

    for (long i = 0; i < CONSUMER_NUM; i++) {
        pthread_create(&consumers[i], NULL, consumerThread, (void*)i);
        sleep(1);
    }

    for (int i = 0; i < PRODUCER_NUM; i++) {
        pthread_join(producers[i], NULL);
    }

    for (int i = 0; i < CONSUMER_NUM; i++) {
        pthread_join(consumers[i], NULL);
    }

    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&condEmpty);
    pthread_cond_destroy(&condFull);

    return 0;
}

