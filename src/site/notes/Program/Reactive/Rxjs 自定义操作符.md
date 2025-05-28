---
{"dg-publish":true,"permalink":"/Program/Reactive/Rxjs 自定义操作符/","noteIcon":"","created":"2024-11-26T14:47:15.452+08:00"}
---

```typescript
import {
  concatWith,
  delay,
  EMPTY,
  filter,
  map,
  Observable,
  of,
  OperatorFunction,
  retry,
  tap,
  scan,
  shareReplay,
  switchMap,
  take,
  throwError,
  isObservable,
  isEmpty,
  concatMap,
} from "rxjs";

/**
 * 创建一个操作符，用于在Observable流中筛选出Falsy值，并在遇到Falsy值时抛出错误。
 * Falsy值包括null和undefined，当源Observable发出这样的值时，将使用提供的错误工厂函数创建一个错误并抛出。
 *
 * @param errorFactory 一个生成错误的工厂函数，当源Observable发出Falsy值时调用。
 * @returns 返回一个函数，该函数接受一个Observable并返回一个新的Observable。
 * 当输入的Observable发出Falsy值时，新的Observable将抛出由errorFactory生成的错误。
 */
export function throwIfFalsy<T>(errorFactory: () => Error) {
  return (source: Observable<T>): Observable<T> => {
    return source.pipe(
      switchMap((value) => {
        // 检查发出的值是否为Falsy（null或undefined）
        if (value === null || typeof value === "undefined") {
          // 如果是Falsy值，则使用提供的错误工厂函数生成一个错误并抛出
          return throwError(errorFactory);
        }
        // 如果不是Falsy值，则继续发出原始值
        return of(value);
      }),
    );
  };
}
/**
 * 创建一个操作符函数，用于过滤掉源Observable中的null或undefined值
 * 这个函数的目的是从Observable流中移除null或undefined的项，确保订阅者只接收到有效（非null和非undefined）的值
 *
 * @returns {OperatorFunction<T, T>} 返回一个操作符函数，它接受一个Observable作为输入，并返回一个新的Observable，
 * 过滤掉了null或undefined值
 */
export function filterNotNull<T>(): OperatorFunction<T, T> {
  return ($source: Observable<T | null | undefined>) =>
    $source.pipe(
      switchMap((value) => {
        if (value !== null && typeof value !== "undefined") {
          return of(value);
        } else {
          return EMPTY;
        }
      }),
    );
}
/**
 * 创建一个缓存操作符，用于缓存Observable的数据流
 *
 * @param size 缓存的大小，默认为1，表示只缓存最近的一个数据
 *
 * 此函数的主要作用是将源Observable转换为一个具有缓存功能的Observable
 * 当订阅者订阅这个转换后的Observable时，即使源数据流已经完成或出错，订阅者也能接收到缓存的数据
 * 这在需要对Observable数据进行复用的场景下特别有用，尤其是在数据获取成本较高或需要在多个订阅者之间共享数据时
 *
 * @returns 返回一个函数，该函数接受一个Observable，并返回一个具有缓存功能的Observable
 */
export function cache<T>(size: number = 1) {
  return (source$: Observable<T>) =>
    source$.pipe(shareReplay({ bufferSize: size, refCount: true }));
}

/**
 * 创建一个带有延迟的重试操作符
 *
 * 该函数用于生成一个RxJS操作符，该操作符可使源Observable在发出错误时按照指定次数和延迟进行重试
 * 当重试次数超过最大值时，会发出一个"Retries exceeded"的通知
 *
 * @param maxRetries 最大重试次数
 * @param delayMs 每次重试之间的延迟时间，单位为毫秒
 * @returns 返回一个函数，该函数接受一个源Observable，并返回一个经过重试策略处理的Observable
 */
export function retryWithDelay<T>(maxRetries: number, delayMs: number) {
  // 返回一个操作符，该操作符对源Observable应用重试逻辑
  return (source$: Observable<T>) =>
    source$.pipe(
      retry({
        // 当源Observable发出错误时，应用带有延迟的重试逻辑
        delay: (errors) =>
          errors.pipe(
            // 在每次错误后延迟指定的毫秒数
            delay(delayMs),
            // 限制重试的最大次数
            take(maxRetries),
            // 当重试次数超过最大值时，发出一个"Retries exceeded"的通知
            concatWith(of("Retries exceeded")),
          ),
      }),
    );
}

export function rateLimiter<T>(limit: number, windowMs: number) {
  return (source$: Observable<T>) =>
    source$.pipe(
      scan((events, _) => {
        const now = Date.now();
        const validEvents = events.filter(
          (timestamp) => now - timestamp < windowMs,
        );
        validEvents.push(now);
        return validEvents.length > limit ? events : [...validEvents];
      }, [] as number[]),
      filter((events) => events.length <= limit),
      map((_, i) => i),
    );
}

/**
 * 根据条件在Observable管道中执行指定的操作
 *
 * 当条件满足时，对Observable的每个发出的值执行next指定的操作如果条件不满足，则不执行任何操作
 * 这个函数的主要用途是在不中断Observable流的情况下，基于某些条件执行副作用操作（如日志记录、事件触发等）
 *
 * @param condition - 可以是一个布尔值或一个返回布尔值的函数，用于判断是否执行next操作
 * @param next - 一个函数，当条件满足时，会对Observable的每个发出的值执行此函数
 * @returns 返回一个高阶Observable，它会根据条件执行next指定的操作
 */
export function tapIf<T>(
  condition: boolean | ((it: T) => boolean),
  next: (it: T) => any,
) {
  return (source$: Observable<T>) => {
    return source$.pipe(
      tap((it) => {
        if (typeof condition === "boolean") {
          if (!condition) return;
        } else {
          if (!condition(it)) return;
        }
        const r = next(it);
        if (isObservable(r)) {
          r.subscribe();
        }
      }),
    );
  };
}

/**
 * 自定义操作符 switchIfEmpty
 * @param alternate$ 替代的 Observable
 * @returns 一个新的 Observable
 */
export function switchIfEmpty<T>(alternate$: Observable<T>) {
  return (source$: Observable<T>): Observable<T> =>
    source$.pipe(
      isEmpty(), // 判断源 Observable 是否为空
      concatMap((isEmpty) => (isEmpty ? alternate$ : source$)), // 根据是否为空切换
    );
}

```