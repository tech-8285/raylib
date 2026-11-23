# Unit tests for raylib
#
# Usage:
#   make -C tests          # build and run all tests
#   make -C tests clean

CC ?= cc
CFLAGS ?= -std=c99 -Wall -Wextra -O1
LDFLAGS ?= -lm -lpthread

TESTS = raudio_stream_playback_test

.PHONY: all clean
all: $(TESTS)
	@for t in $(TESTS); do ./$$t || exit 1; done

raudio_stream_playback_test: raudio_stream_playback_test.c ../src/raudio.c
	$(CC) $(CFLAGS) -I../src $< -o $@ $(LDFLAGS)

clean:
	rm -f $(TESTS)
