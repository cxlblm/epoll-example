#include <stdio.h>
#include <sys/epoll.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <unistd.h>  
#include <netinet/in.h>
#include <fcntl.h>  
#include <netdb.h>
#include <string.h>  


#define EPOLL_FLAG 0
#define MAX_CONN 128
#define EVENTS_NUM 100

int set_non_block(int socket)
{
    int flag = fcntl(socket, F_GETFL, 0);
    if (flag < 0) {
        return flag;
    }
    return fcntl(socket, F_SETFL, flag | O_NONBLOCK);
}

int create_socket(unsigned short port)
{
    int fd = socket(AF_INET6, SOCK_STREAM | SOCK_NONBLOCK, IPPROTO_TCP);
    if (fd < 0) {
        return fd;
    }
    struct sockaddr_in6 address;
    memset(&address, 0, sizeof(address));
    address.sin6_family= AF_INET6;
    address.sin6_port = htons(port);     // 需要网络序, 也就是大端
    // inet_pton(AF_INET6, "::", &address.sin6_addr);
	address.sin6_addr = in6addr_any;
    int r = bind(fd, (struct sockaddr*)&address, sizeof(struct sockaddr_in6));
    if (r < 0) {
        return r;
    }
    int error = listen(fd, MAX_CONN);
    if (error < 0) {
        return error;
    }
    return fd;
}

int init_epoll(int sfd, struct epoll_event *event)
{
    int epoll_fd = epoll_create1(EPOLL_FLAG);
    if (epoll_fd < 0) {
        return epoll_fd;
    }

    event->data.fd = sfd;
    int s = epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sfd, event);
    if (s < 0) {
        return s;
    }
    return epoll_fd;
}

int init_events(struct epoll_event **events)
{
    *events = calloc(EVENTS_NUM, sizeof(struct epoll_event));
    if (NULL == *events) {
        return -1;
    }
    return 0;
}

int socket_accept(int socket, int epoll_fd, struct epoll_event *event)
{
    struct sockaddr_in6 address;
    socklen_t len = sizeof(address);
    memset(&address, 0, sizeof address);
    int infd = accept(socket, (struct sockaddr*) &address, &len);
    if (infd < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            return infd;
        }
    }
    int block_error;
    block_error = set_non_block(infd);
    if (block_error < 0) {
        return block_error;
    }

    char host_buf[INET6_ADDRSTRLEN];
    // inet_ntop(AF_INET6, &host_buf, sizeof(host_buf), sizeof(address));
    // printf("Accepted connection on descriptor %d " "(host=%s, port=%d)\n", infd, host_buf, address.sin6_port);

    block_error = set_non_block(infd);
    if (block_error < 0) {
        return block_error;
    }
    event->data.fd = infd;
    event->events = EPOLLOUT|EPOLLIN|EPOLLET;
    return epoll_ctl(epoll_fd, EPOLL_CTL_ADD, infd, event);
}

ssize_t read_conn(int conn, char *buffer, size_t size)
{
    ssize_t count;
    count = read(conn, buffer, size);
    if (count < 0) {
        if (errno != EAGAIN) {
            // 需要文件句柄, 否则就会出错
            return 0;
        }
        return count;
    }
    return count;
}

void write_screen(int epoll_fd, struct epoll_event *event)
{
    char buffer[256];
    size_t buffer_size = 256;
    while (1) {
        ssize_t count = read_conn(event->data.fd, (void *)&buffer, buffer_size);
        if (count < 0) {
            if (errno == EAGAIN) {
                perror("conn read EAGAIN ");
                break;
            }
            perror("conn read error ");
            int r = epoll_ctl(epoll_fd, EPOLL_CTL_DEL, event->data.fd, event);
            if (r < 0) {
                perror("epoll error conn");
                break;
            }
            close(event->data.fd);
            // 还需要关闭数据
            break;
        }
        int r = write(1, buffer, count);
        if (r < 0) {
            perror("write error conn ");
            break;
        }
    }
}

void wait(int epoll_fd, int socket)
{
    struct epoll_event *events;
    int error = init_events(&events);
    if (error < 0) {
        return;
    }
    while (1) {
        int n = epoll_wait(epoll_fd, events, EVENTS_NUM, -1);
        for (int i = 0; i < n; ++i) {
            struct epoll_event local_event;
            local_event = events[0];
            if (local_event.events & (EPOLLERR | EPOLLHUP)) {
                fprintf(stderr, "epoll error");
                int r = epoll_ctl(epoll_fd, EPOLL_CTL_DEL, local_event.data.fd, &local_event);
                if (r > 0) {
                    close(local_event.data.fd);
                }
            } else if (local_event.events & EPOLLIN) {
                if (local_event.data.fd == socket) {    // 说明是accept请求
                    int r = socket_accept(socket, epoll_fd, &local_event);
                    if (r < 0) {
                        perror("accept error");
                        // 如果出错这里是否要退出
                        continue;
                    }
                } else {        // 说明是有数据传递过来
                    write_screen(epoll_fd, &local_event);
                }

            } else if (local_event.events & EPOLLOUT) {
				write(local_event.data.fd, "Welcome\n", 8);
				local_event.events = EPOLLIN | EPOLLET;
				epoll_ctl(epoll_fd, EPOLL_CTL_MOD, local_event.data.fd, &local_event);
            }
        }
    }
}

int main(void)
{
    int socket = create_socket(8888);
    if (-1 == socket) {
        perror("socket error ");
        return 0;
    }

    struct epoll_event event;
    event.events = EPOLLIN | EPOLLET;

    int epoll_fd = init_epoll(socket, &event);
    if (epoll_fd < 0) {
        perror("epoll error");
        return 0;
    }
    wait(epoll_fd, socket);

    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, socket, &event);
    close(epoll_fd);
}
