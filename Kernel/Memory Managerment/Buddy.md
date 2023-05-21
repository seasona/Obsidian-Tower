Memory managerment is one of most complex subsystem in linux kernel, it's very difficult to figure out panorama.

This article is my analyze of buddy. Linux kernel version is 5.10

## Zone

At first, memoy is managed by zone:

- ZONE_DMA

	Contains page frames of memory below 16 MB

- ZONE_NORMAL

	Contains page frames of memory at and above 16 MB and below 896 MB

- ZONE_HIGHMEM

	Contains page frames of memory at and above 896 MB

In 64bit system, ZONE_HIGHMEM is no use, we care about ZONE_NORMAL mostly.

`struct zone` is defined in *mmzone.h*，this struct is huge so we just skip it.

## free_area

we care about the most import part: `struct free_area`

```c
struct free_area {
	struct list_head	free_list[MIGRATE_TYPES];
	unsigned long		nr_free;
}
```

member `free_list` store free page, and member `nr_free` indicate left free page number.

we can get free page by getting first entry of free_list simply.

```c
static inline struct page *get_page_from_free_area(struct free_area *area,
					    int migratetype)
{
	return list_first_entry_or_null(&area->free_list[migratetype],
					struct page, lru);
}
```

Look at function `get_page_from_free_area`，we can find `struct page` is connected by member lru

## How to get a page

We can understand this get page process clearly by diving into `__rmqueue_smallest` function:

```c
static __always_inline
struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
								int migratetype)
{
	unsigned int current_order;
	struct free_area *area;
	struct page *page;

	/* Find a page of the appropriate size in the preferred list */
	for (current_order = order; current_order < MAX_ORDER; ++current_order) {
		area = &(zone->free_area[current_order]);
		page = get_page_from_free_area(area, migratetype);
		if (!page)
			continue;
		del_page_from_free_list(page, zone, current_order);
		expand(zone, page, order, current_order, migratetype);
		set_pcppage_migratetype(page, migratetype);
		return page;
	}

	return NULL;
}
```

Firstly, we will give a order which indicate continuous memory size (1, 2, 4, 8, 16x page size). And we walk through free_area[order]

Then, we get struct page from free_list if free_list is not empty, if so, we delete this page from free_list and reduce free_area[order].nr_free.

```c
static inline void del_page_from_free_list(struct page *page, struct zone *zone,
										   unsigned int order)
{
	/* clear reported state and update reported page count */
	if (page_reported(page))
		__ClearPageReported(page);

	list_del(&page->lru);
	__ClearPageBuddy(page);
	set_page_private(page, 0);
	zone->free_area[order].nr_free--;
}
```

Finally, if we can't get page from list of this memory size, we will look from bigger memory size list.

![[Buddy.png]]

## Mapping between pfn and page


## Reference

- https://www.byteisland.com/linux-%E5%86%85%E6%A0%B8-buddy-%E7%B3%BB%E7%BB%9F/
- https://gohalo.me/post/kernel-memory-management-from-kernel-view.html