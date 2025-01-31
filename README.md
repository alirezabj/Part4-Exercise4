# Part4-Exercise4




### A and B)

**Modify Zipper to use generics**

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

    /**
     * Executes unzipping and creates a handler for every created file.
     *
     * @return A list of processed objects of type X.
     * @throws IOException In case of any I/O errors.
     */
    public List<X> run() throws IOException {
        unzip();
        return createHandlers().stream().map(Handler::handle).toList();
    }

    /**
     * Creates handlers for all files in the extracted folder.
     *
     * @return A list of handlers for processing the extracted files.
     * @throws IOException In case of any I/O errors.
     */
    protected List<Handler<X>> createHandlers() throws IOException {
        try (final var stream = Files.list(tempDirectory)) {
            return stream.map(this::createHandler).toList();
        }
    }

    /**
     * Creates a handler for a given file.
     *
     * @param file The file to be processed.
     * @return A handler for processing the file.
     */
    protected abstract Handler<X> createHandler(Path file);

    /**
     * A generic handler for processing extracted files.
     *
     * @param <X> The type of result returned after processing the file.
     */
    protected abstract static class Handler<X> {
        protected final Path file;

        /**
         * Initializes the handler with a file.
         *
         * @param file The file to be handled.
         */
        public Handler(Path file) {
            this.file = file;
        }

        /**
         * Processes the file and returns the result.
         *
         * @return The processed result of type X.
         */
        public abstract X handle();
    }
}

```
