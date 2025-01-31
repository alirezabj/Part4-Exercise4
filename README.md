# Part4-Exercise4




### A and B)

**Modifying Zipper to use generics**

```java
package fi.utu.tech.ooj.exercise4.exercise4;

import fi.utu.tech.ooj.exercise4.Main;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Comparator;
import java.util.List;
import java.util.zip.ZipInputStream;


abstract public class Zipper<X> implements AutoCloseable {
    // zip-file for unzipping
    private final String zipFile;

    // java class, from which package the zip file is looked for
    private final Class<?> resolver = Main.class;

    // path of temp directory
    protected final Path tempDirectory;

    /**
     * Records the given zip file and
     * creates a temporary directory 'tempDirectory'.
     *
     * @param zipFile Zip file path (precondition: must exist and be non-null).
     * @throws IOException If the zip file is not found or the directory cannot be created.
     */
    public Zipper(String zipFile) throws IOException {
        // WORKAROUND: if the zip file is not found, comment out the next two lines.
        if (resolver.getResource(zipFile) == null)
            throw new FileNotFoundException(zipFile);

        this.zipFile = zipFile;
        tempDirectory = Files.createTempDirectory("dtek0066");
        System.out.println("Created a temp directory " + tempDirectory);
    }

    /**
     * Deletes the temporary directory 'tempDirectory' when the object is closed.
     *
     * @throws IOException In case of any I/O errors.
     */
    @Override
    public void close() throws IOException {
        try (final var stream = Files.walk(tempDirectory)) {
            stream.sorted(Comparator.reverseOrder())
                    .map(Path::toFile)
                    .forEach(File::delete);
        }
        System.out.println("The removed temp directory was " + tempDirectory);
    }

    /**
     * Unzip the file 'zipFile' to the temporary directory 'tempDirectory'
     *
     * @throws IOException In case of any I/O errors.
     */
    private void unzip() throws IOException {
        final var destinationDir = tempDirectory.toFile();

        // WORKAROUND: If the zip file is not found, change to the following
        // try (final var inputStream = new FileInputStream(zipFile);
        try (final var inputStream = resolver.getResourceAsStream(zipFile);
             final var stream = new ZipInputStream(inputStream)) {
            var zipEntry = stream.getNextEntry();
            while (zipEntry != null) {
                final var newFile = new File(destinationDir, zipEntry.getName());
                final var destDirPath = destinationDir.getCanonicalPath();
                final var destFilePath = newFile.getCanonicalPath();
                if (!destFilePath.startsWith(destDirPath + File.separator)) {
                    throw new IOException("Entry is outside of the target dir: " + zipEntry.getName());
                }
                System.out.println("Puretaan " + newFile);
                if (zipEntry.isDirectory()) {
                    if (!newFile.isDirectory() && !newFile.mkdirs()) {
                        throw new IOException("Failed to create directory: " + newFile);
                    }
                } else {
                    // fix for Windows-created archives
                    final var parent = newFile.getParentFile();
                    if (!parent.isDirectory() && !parent.mkdirs()) {
                        throw new IOException("Failed to create directory: " + parent);
                    }

                    // write file content
                    try (final var fos = new FileOutputStream(newFile)) {
                        stream.transferTo(fos);
                    }
                }
                zipEntry = stream.getNextEntry();
            }
            stream.closeEntry();
        }
    }

    /*
     * executes unzipping and creates a handler for every created file
     * @return a list of processed objects of type X
     * @throws IOException In case of any I/O errors
     */
    public List<X> run() throws IOException {
        unzip();
        return createHandlers().stream().map(Handler::handle).toList();
    }

    /**
     * creates handlers for all files in the extracted folder
     * @return a list of handlers for processing the extracted files
     * @throws IOException In case of any I/O errors
     */
    protected List<Handler<X>> createHandlers() throws IOException {
        try (final var stream = Files.list(tempDirectory)) {
            return stream.map(this::createHandler).toList();
        }
    }

    /**
     * creates a handler for a given file
     * @param file The file to be processed
     * @return A handler for processing the file
     */
    protected abstract Handler<X> createHandler(Path file);

    /**
     * a generic handler for processing extracted files
     * @param <X> The type of result returned after processing the file
     */
    protected abstract static class Handler<X> {
        protected final Path file;

        /*
         * Initializes the handler with a file
         *
         * @param file The file to be handled 
         */
        public Handler(Path file) {
            this.file = file;
        }

        /**
         * Processes the file 
         * @return The processed result of type X
         */
        public abstract X handle();
    }
}

```

**Changes:**
- Added generics (X) to Zipper and Handler. Therefore, instead of modifyinh an external list, each handler returns an object of type X
- Updated run() to return a list of results. Now, run() collects the results from all handlers and returns them as a List<X>.
- Updated Handler class to be generic. Each handler now returns an object instead of just processing files.

--

**Modifying TestZipper to use generics**

```java
package fi.utu.tech.ooj.exercise4.exercise1;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.regex.Pattern;

// use generics but returns no meaningful data
class TestZipper extends Zipper<Void> {  
    TestZipper(String zipFile) throws IOException {
        super(zipFile);
    }

    @Override
    public List<Void> run() throws IOException {
         // process files but return an empty list
        return super.run();
    }

    @Override
    protected Handler<Void> createHandler(Path file) {
        return new Handler<>(file) {
            @Override
            public Void handle() throws IOException {
                var regex = Pattern.compile("\\W");
                var contents = Files.readString(file);
                var lines = Files.readAllLines(file);
                var firstLine = lines.isEmpty() ? "unknown" : lines.getFirst();
                var words = regex.splitAsStream(contents)
                        .filter(s -> !s.isBlank())
                        .map(String::toLowerCase)
                        .toList();

                System.out.printf("""
                                
                                Originally was fetched from %s.
                                The founded file is %s.
                                The file contains %d lines.
                                The file contains %d words.
                                Possible title of the work: %s
                                
                                """,
                        tempDirectory,
                        file.getFileName(),
                        lines.size(),
                        words.size(),
                        firstLine
                );

                // must return null because Void cannot have a value
                return null; 
            }
        };
    }
}

```

**Changes:**
- extends Zipper<Void> - using generics but does not return data since void means nothig
- List<Void> run()	- calling `super.run()` but returns an empty list since Void is used
- Handler<Void> - the handler still processes files but does not return anything useful
- returning null - since Void is used we must return null explicitly

--


**Modifying TestZipper2 to use generics**

```java
package fi.utu.tech.ooj.exercise4.exercise2;

import fi.utu.tech.ooj.exercise4.exercise1.Zipper;
import java.io.IOException;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

class Book {
}

class TestZipper2 extends Zipper<Book> {
    private final List<Book> books = new ArrayList<>();
    private int idx = 0;

    TestZipper2(String zipFile) throws IOException {
        super(zipFile);
    }

    @Override
    public void run() throws IOException {
        super.run();

        System.out.printf("""

                Handled %d Books.
                Now we could sort it out a bit.

                """, idx); 
    }

    @Override
    protected Handler<Book> createHandler(Path file) {
        return new Handler<>(file) {
            @Override
            public Book handle() {
                Book book = new Book();
                books.add(book); 
                idx++; 
                return book;
            }
        };
    }
}


```

**Changes:**
- TestZipper2 extends Zipper<Book> - `TestZipper2` is genenric and now it works with `Book` objects explicitly 
- replacing Book[] books with List<Book> books	- allowing dynamic handling of books without a fixed array size
- changing run() method return type from void to List<Book> - now it returns a list of processed books instead of modifying a field directly
- updating createHandler() to return Handler<Book> to ensure handlers return Book objects
- handle() to return a Book object - each handler now creates and returns a Book

### C)

**Define the sorting interface**
```java
package fi.utu.tech.ooj.exercise4.exercise4;

import java.util.List;

/*
 * define a generic sorting mechanism for books
 */
@FunctionalInterface
public interface BookSorter {
    /*
     * sorts a list of books based on a specific criterion.
     * @param books The list of books to sort.
     * @return A new sorted list of books.
     */
    List<Book> sort(List<Book> books);
}

```
- This interface allows different sorting strategies to be implemented without modifying the existing code. This interface is also generic because it does not specify how the books should be sorted leaving it open for different implementations.

**Define a method to sort and print books**

```java
package fi.utu.tech.ooj.exercise4.exercise4;

import java.util.List;

/*
 * utility class to handle book sorting operations
 */
public class BookCollectionHandler {
    /**
     * sort and print books based on the given sorting strategy
     * @param books The list of books to sort
     * @param sorter The sorting strategy to apply
     */
    public static void processAndPrintBooks(List<Book> books, BookSorter sorter) {
        List<Book> sortedBooks = sorter.sort(books);
        sortedBooks.forEach(System.out::println);
    }
}

```
- This method ensures that sorting and printing are handled in a single and reusable function.


