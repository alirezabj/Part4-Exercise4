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
import java.util.List;

class Book {
}

class TestZipper2 extends Zipper<Book> { 
    
    private Book[] books = new Book[100];  
    private int idx = 0;

    TestZipper2(String zipFile) throws IOException {
        super(zipFile);
    }

    @Override
    public void run() throws IOException {
        List<Book> processedBooks = super.run(); 
        
        for (Book book : processedBooks) {
            if (idx < books.length) {
                books[idx++] = book;
            }
        }

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
                return new Book();   
            }
        };
    }
}


```

**Changes:**
- TestZipper2 extends Zipper<Book> - `TestZipper2` is genenric and now it works with `Book` objects explicitly 
- capture the returned `List<Book>` from `super.run()` - The run() method now captures the list of processed Book objects returned from the super.run() call.
- add processed books to the `books` array
- use genenrics in `Handler<Book>`
- `handle()` returnes a book object.

### C)
This is the code that I had for Exercise 4 Part 2:

**`Book`**

```java

package fi.utu.tech.ooj.exercise4.exercise2;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Set;
import java.util.regex.Pattern;
import java.util.stream.Collectors;


public class Book implements Comparable<Book> {
    // first line of the file
    private final String name;  
    //total number of lines in the file
    private final int lineCount;
    // unique word count in the file
    private final int uniqueWordCount;  


     /*
     * making a Book object by reading the file content
     * @param filePath the path to the file
     * @throws IOException if an error occurs while reading the file (if it does not exist)
     */
    public Book(Path filePath) throws IOException {
        //read the book content from a file
        List<String> lines = Files.readAllLines(filePath);
        // check if the file contains content. if it contains, the book's name is set as the first line of the file otherwise if the file is empty, it returns No title
        this.name = lines.isEmpty() ? "No Title" : lines.get(0);
       // count the total number of lines
        this.lineCount = lines.size();

        // compute unique words
        Pattern wordPattern = Pattern.compile("\\W+");
        this.uniqueWordCount = lines.stream()
                .flatMap(line -> wordPattern.splitAsStream(line)) // split into words
                .filter(word -> !word.isBlank()) // remove empty strings
                .map(String::toLowerCase) // convert to lowercase 
                .collect(Collectors.toSet()) // store in a Set to remove duplicates
                .size();
    }


     /*
     * return the book's first line of the file
     */
    public String getName() {
        return name;
    }

    /*
     * return the number of lines in the book
     */
    public int getLineCount() {
        return lineCount;
    }

    /*
     * return the total number of unique word 
     */
    public int getUniqueWordCount() {
        return uniqueWordCount;
    }

     /*
     * define the natural order of books based on the alphabetical order of their names
     * @param other the other book to compare with
     * @return negative if this book's name comes first, positive if it comes later, 0 if equal
     */
    @Override
    public int compareTo(Book other) {
        return this.name.compareTo(other.name);
    }

    /*
    * return a string with the book's title, line count, and unqiur word count
    */
    @Override
    public String toString() {
        return "Book: '" + name + "' | Lines: " + lineCount + " | Unique Words: " + uniqueWordCount;

    }
}
```

**`TestZipper2`**
```java
package fi.utu.tech.ooj.exercise4.exercise2;

import fi.utu.tech.ooj.exercise4.exercise1.Zipper;

import java.io.IOException;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;


public class TestZipper2 extends Zipper {
    
    private final List<Book> books = new ArrayList<>();

    public TestZipper2(String zipFile) throws IOException {
        super(zipFile);
    }


    @Override
    public void run() throws IOException {
        super.run();

        // sort books in natural order 
        books.sort(Comparator.naturalOrder());

        // print books sorted by name
        for (Book book : books) {
            System.out.println(book);
        }

        // sort books by line count in ascending order
        books.sort(Comparator.comparingInt(Book::getLineCount));


        // print books sorted by line count
        for (Book book : books) {
            System.out.println(book);
        }

        // sort books by unique word count in descending order
        books.sort(Comparator.comparingInt(Book::getUniqueWordCount).reversed());

        // print books sorted by unique word count
        for (Book book : books) {
            System.out.println(book);
        }
    }


    @Override
    protected Handler createHandler(Path file) {
        return new Handler(file) {
            @Override
            public void handle() {
                try {
                   // add new Book to the list
                    books.add(new Book(file));
                } catch (IOException e) {
                    System.err.println("Failed to read book");
                }
            }
        };
    }
}
```

**Interface:**
```java
```







