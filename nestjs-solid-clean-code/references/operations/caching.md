# Caching

## Core Idea

Caching stores the result of an expensive operation (a database query, a call to an external API) and returns it again while the data is still valid — instead of doing the same work over on every request.

## `@nestjs/cache-manager` with Redis

```ts
@Module({
  imports: [
    CacheModule.registerAsync({
      useFactory: () => ({
        store: redisStore,
        host: 'localhost',
        port: 6379,
        ttl: 60, // seconds
      }),
    }),
  ],
})
export class AppModule {}
```

### Automatic Caching via Interceptor

```ts
@UseInterceptors(CacheInterceptor)
@CacheTTL(30)
@Get('popular-products')
getPopularProducts() {
  // the result is cached automatically, keyed by URL
}
```

### Manual Cache Management

For finer control (a custom key, invalidation on data change), the cache is used directly through a service:

```ts
@Injectable()
export class ProductService {
  constructor(
    @Inject(CACHE_MANAGER) private readonly cache: Cache,
    private readonly productRepository: ProductRepository,
  ) {}

  async getById(id: number) {
    const cacheKey = `product:${id}`;
    const cached = await this.cache.get<Product>(cacheKey);
    if (cached) return cached;

    const product = await this.productRepository.findOne(id);
    await this.cache.set(cacheKey, product, { ttl: 300 });
    return product;
  }

  async update(id: number, dto: UpdateProductDto) {
    const product = await this.productRepository.update(id, dto);
    await this.cache.del(`product:${id}`); // invalidate on data change
    return product;
  }
}
```

## Invalidation Strategies

- **TTL (time-to-live)** — the simplest option: a cache entry "expires" by itself after N seconds. Suitable when a small delay in freshness is acceptable.
- **Explicit invalidation on write** — the cache is deleted or updated right after the data changes (as in the example above). More precise, but requires remembering to add invalidation everywhere the data can change.
- **Cache-aside** (the pattern shown above: check the cache first, and on a miss, go to the database and put the result in the cache) — the most common approach for reads.

## Where Caching Helps and Where It's Dangerous

**Good to cache:** data that changes rarely and is expensive to compute (a product catalog, public content, aggregated reports).

**Dangerous to cache without careful invalidation:** financial balances, order status, any data where staleness could lead to a wrong business decision (for example, showing a product as "in stock" when it has already sold out). In such cases, either the TTL must be very short, or caching isn't worth the risk at all — again, a question of YAGNI (`references/principles/yagni.md`): caching is justified only when there's an actual performance problem it solves.

## Why This Matters

Caching is an optimization, not an architectural principle: it adds another layer of state (cached data can drift from the real data), so it's introduced deliberately, for a specific performance bottleneck, not "just in case" on every endpoint.

<!-- Open for expansion: distributed cache vs. in-memory cache in a multi-instance application, cache stampede and how to protect against it -->
