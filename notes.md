# Secure Coding Guidelines for Java SE
[ref](http://www.oracle.com/technetwork/java/seccodeguide-139067.html)

# Denied of Service
## Guideline 1-1 / DOS-1: Beware of activities that may use disproportionate resources
- Examples of attacks include:
  - Requesting a large image size for vector graphics. For instance, SVG and font files
  - Integer overflow errors can cause sanity checking of sizes to fail.
  - An object graph constructed by parsing a text or binary stream may have memory requirements many times that of the original data
  - "Zip bombs" whereby a short file is very highly compressed. For instance, ZIPs, GIFs and gzip encoded HTTP contents. When decompressing files it is better to set limits on the decompressed data size rather than relying upon compressed size or meta-data.
  - "Billion laughs attack" whereby XML entity expansion causes an XML document to grow dramatically during parsing. Set the XMLConstants.FEATURE_SECURE_PROCESSING feature to enforce reasonable limits.
  - Causing many keys to be inserted into a hash table with the same hash code, turning an algorithm of around O(n) into O(n2).
  - XPath expressions may consume arbitrary amounts of processor time.
  - Regular expressions may exhibit [catastrophic backtracking](https://www.regular-expressions.info/catastrophic.html).
  - Java deserialization and Java Beans XML deserialization of malicious data may result in unbounded memory or CPU usage.
  - Detailed logging of unusual behavior may result in excessive output to log files.
  - Infinite loops can be caused by parsing some corner case data. Ensure that each iteration of a loop makes some progress.

## Guideline 1-2 / DOS-2: Release resources in all cases
- Java SE 8 lamda feature
```
long sum = readFileBuffered(InputStream in -> {
    long current = 0;
    for (;;) {
        int b = in.read();
        if (b == -1) {
            return current;
        }
        current += b;
    }
});
```

- Try with resources syntax in Java SE 7
```
public  R readFileBuffered(
    InputStreamHandler handler
) throws IOException {
    try (final InputStream in = Files.newInputStream(path)) {
        handler.handle(new BufferedInputStream(in));
    }
}
```

- For resources without support for the enhancement feature, use the standard resource acquisition and release
```
public  R locked(Action action) {
    lock.lock();
    try {
        return action.run();
    } finally {
        lock.unlock();
    }
}
```

- Ensure that any output buffers are flushed in the case that output was otherwise successful. If the flush fails, the code should exit via an exception.
```
public void writeFile(
    OutputStreamHandler handler
) throws IOException {
    try (final OutputStream rawOut = Files.newOutputStream(path)) {
        final BufferedOutputStream out =
            new BufferedOutputStream(rawOut);
        handler.handle(out);
        out.flush();
    }
}
```

- Some decorators of resources may themselves be resources that require correct release
```
public void bufferedWriteGzipFile(
    OutputStreamHandler handler
) throws IOException {
    try (
        final OutputStream rawOut = Files.newOutputStream(path);
        final OutputStream compressedOut =
                                new GzipOutputStream(rawOut);
    ) {
        final BufferedOutputStream out =
            new BufferedOutputStream(compressedOut);
        handler.handle(out);
        out.flush();
    }
}
```

## Guideline 1-3 / DOS-3: Resource limit checks should not suffer from integer overflow
- Some checking can be rearranged so as to avoid overflow. With large values, `current + max` could overflow to a negative value, which would always be less than `max`.
```
private void checkGrowBy(long extra) {
    if (extra < 0 || current > max - extra) {
          throw new IllegalArgumentException();
    }
}
```

- If performance is not a particular issue, a verbose approach is to use arbitrary sized integers.
```
private void checkGrowBy(long extra) {
    BigInteger currentBig = BigInteger.valueOf(current);
    BigInteger maxBig     = BigInteger.valueOf(max    );
    BigInteger extraBig   = BigInteger.valueOf(extra  );

    if (extra < 0 ||
        currentBig.add(extraBig).compareTo(maxBig) > 0) {
          throw new IllegalArgumentException();
    }
}
```
