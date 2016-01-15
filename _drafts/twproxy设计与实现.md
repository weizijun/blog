###mbuf结构与使用分析

以下是mbuf的结构体

		struct mbuf {
		    uint32_t           magic;   /* mbuf magic (const) */
		    STAILQ_ENTRY(mbuf) next;    /* next mbuf */
		    uint8_t            *pos;    /* read marker */
		    uint8_t            *last;   /* write marker */
		    uint8_t            *start;  /* start of buffer (const) */
		    uint8_t            *end;    /* end of buffer (const) */
		};
		
mbuf在内存中是这样的结构

    /*
     * mbuf header is at the tail end of the mbuf. This enables us to catch
     * buffer overrun early by asserting on the magic value during get or
     * put operations
     *
     *   <------------- mbuf_chunk_size ------------->
     *   +-------------------------------------------+
     *   |       mbuf data          |  mbuf header   |
     *   |     (mbuf_offset)        | (struct mbuf)  |
     *   +-------------------------------------------+
     *   ^           ^        ^     ^^
     *   |           |        |     ||
     *   \           |        |     |\
     *   mbuf->start \        |     | mbuf->end (one byte past valid bound)
     *                mbuf->pos     \
     *                        \      mbuf
     *                        mbuf->last (one byte past valid byte)
     *
     */
     
     
一个mbuf结构体总长度是mbuf_chunk_size，mbuf_chunk_size可以通过启动时设置-m参数设置，默认是`16K`字节，-m设置最小`512`字节，最大`16M`字节。