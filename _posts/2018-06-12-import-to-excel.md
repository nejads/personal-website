---
layout: page
title: Import to Excel functionality in Spring framework
permalink: /import-to-excel/
---

In this article we will see how to implement a back-end service in springboot applications to get some data in Excel format and serve it to the client. Generating huge excel file is CPU- and memory demanding. In other side, dependening on which version of excel will you generate, the file has limit e.g. Excel 2003 has max capacity on 65,535 rows. The chart below shows some facts about Excel file capacity. For full description on different version of Excel, pleas check [this answer on StachExchange][se].
|  |  Max. Rows  | Max. Columns | Max. Cols by letter |
| ------ | ------ | ------ | ------ |
| Excel 365*      | 1,048,576 | 16,384       | XFD                 |
| Excel 2013      | 1,048,576 | 16,384       | XFD                 |
| Excel 2010      | 1,048,576 | 16,384       | XFD                 |
| Excel 2007      | 1,048,576 | 16,384       | XFD                 |
| Excel 2003      | 65,536    | 256          | IV                  |
| Excel 2002 (XP) | 65,536    | 256          | IV                  |
| Excel 2000      | 65,536    | 256          | IV                  |
| Excel 97        | 65,536    | 256          | IV                  |
| Excel 95        | 16,384    | 256          | IV                  |
| Excel 5         | 16,384    | 256          | IV                  |

We as developer must be aware of those limitations and validate a request and if it would go over the boundries, cast out an fault message....
In this solution I will present reactive programming in writing REST APIs and use streams to get result to the client. In this manner I'll try to reach a good enough performance in an enterprice application.

The most important lesson for me in implementing this kind of service was the performace of the query in native query way and non-native query like using ORM. With usign native query I reduced query time significant from 12 seconds to 0.34 second.

## A Use case
Imagine we want to list a bookstore's books in a period of time and import this information to Excel file. Each book can has one or more writers and it belongs to a bookstore. In the query we will specify startdate and  stopdate. The book's publish date shall place in that period.

### How to implement
My simple endpoint looks like as below. I definitly want to handle this kind of requests asynchronously. I do not want to this heavy requests stop the functionality of my web service.

```sh
@Component
@Path("/v1")
public class BookApi {
    private final BookstoreObservables bookstoreObservables;

    @Autowired
    public BookApi(BookstoreObservables bookstoreObservables) {
        this.bookstoreObservables = bookstoreObservables;
    }

    @GET
    @ManagedAsync
    @Produces({"application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"})
    @Path("/bookstore/{bookstoresId}/books/export")
    public void getBookExcelByBookstoresId(
        @Context HttpServletRequest request,
        @Suspended AsyncResponse response,
        @Min(value = 1) @PathParam("bookstoresId") Long bookstoresId,
        @QueryParam("writer") List<String> writers,
        @QueryParam("startPeriod") String startPeriod,
        @QueryParam("stoppPeriod") String stoppPeriod) {

        doInContext(request, response, ctx ->
                bookstoreObservables.getBookExcel(ctx, bookstoresId, writers, startPeriod, stoppPeriod)
                        .map(resp -> Response.status(Response.Status.OK).entity(resp.getDokument())
                                .type(StringUtils.isNotBlank(resp.getMimeTyp()) ?
                                        resp.getMimeTyp() : MimeTypeUtils.ALL_VALUE).build())
                        .onErrorResumeNext(t -> Single.error(t instanceof ErrorException ? translateFieldErrors((ErrorException)t, exceptionFieldReplacements) : t))
        );
    }
}
```

Middle layer to create DTOs and validate incoming request before service layer.

```sh
@Component
public class BookstoreObservables {

    private BookExcelService bookExcelService;

    @Autowired
    public BookExcelService(BookRepository bookExcelService) {
        this.bookExcelService = bookExcelService;
    }

    public Single<GetBookExcelResponse> getBookExcel(ApplicationContext ctx, Long bookstoresId, List<Long> writers, String startPeriod, String stoppPeriod) {

        // ValidatemessageAndBreakOnErrors();
        GetBookExcelDto getBookExcelDto = new GetBookExcel();
        getBookExcelDto.setBookstoresId(bookstoresId);
        getBookExcelDto.setPeriodStart(startPeriod);
        getBookExcelDto.setPeriodStop(stoppPeriod);
        writers.forEach(writer -> getBookExcelDto.getWriters.add(writer));

        return bookExcelService.processMessage(ctx, getBookExcelDto);
    }

}
```
Service layer which use Apache poi to generate Excel file and strem it to upper layer.

```sh
@Service
public class BookExcelService {
    private static final int MAX_NUMBER_OF_ROWS = 10000;
    private BookRepository bookRepository;

    @Autowired
    public BookExcelService(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }


    public GetBookExcelResponse processMessage(GetBookExcelDto req) {

        ByteArrayOutputStream outputStream = null;

        try {
            outputStream = new ByteArrayOutputStream();
            getBooksAsExcelFile(
                    outputStream,
                    req.getBookstoresId(),
                    req.getWritersId(),
                    req.getPeriodStart(),
                    req.getPeriodStop());

        } catch (IOException e) {
            e.printStackTrace();

        } finally {
            if(outputStream != null) {
                try {
                    outputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

    }

    private void getBooksAsExcelFile(OutputStream outputStream, long bookstoresId, List<String> writers,
                                            Period periodStart, Period periodEnd) throws IOException {

        List<BookExcel> books = BookRepository.findBooks(bookstoresId, writersId, periodStart, periodEnd);

        if(books.size() > MAX_NUMBER_OF_ROWS) {
            throw new Exception(String.format(
                    "The query result is more than maximum allowed rows %s", MAX_NUMBER_OF_ROWS));
        }

        SXSSFWorkbook workbook = new SXSSFWorkbook();
        SXSSFSheet sheet = workbook.createSheet("Bookstore");

        createParameters(workbook, bookstoresId, periodStart, periodEnd);
        createHeader(workbook);
        sheet.createFreezePane(0, 5);
        createBooks(workbook, books);

        workbook.write(outputStream);
        workbook.dispose();
    }


    private void createParameters(SXSSFWorkbook workbook, long bookstoresId, Period periodStart, Period periodEnd) {
        SXSSFSheet sheet = workbook.getSheetAt(0);
        int rownum = 0;

        BookstoreDto bookstore = bookstoreService.getBookstoreByBookstoresId(bookstoresId);

        Row r = sheet.createRow(rownum++);
        r.createCell(0, CellType.STRING).setCellValue("Bookstore:");
        r.createCell(1, CellType.STRING).setCellValue(foretag.getNamn());

        r = sheet.createRow(rownum++);
        r.createCell(0, CellType.STRING).setCellValue("Periods:");
        r.createCell(1, CellType.STRING).setCellValue(periodStart.getValue() + " - " + periodEnd.getValue());
    }

    private void createHeader(SXSSFWorkbook workbook) {
        SXSSFSheet sheet = workbook.getSheetAt(0);
        Row header = sheet.createRow(4);

        header.createCell(0, CellType.STRING).setCellValue("Book");
        sheet.setColumnWidth(0, 14 * 256);

        header.createCell(2, CellType.STRING).setCellValue("Period");
        sheet.setColumnWidth(2, 8 * 256);

        header.createCell(5, CellType.STRING).setCellValue("BooksPublishdatum");
        sheet.setColumnWidth(5, 18 * 256);

        header.createCell(6, CellType.STRING).setCellValue("Description");
        sheet.setColumnWidth(6, 22 * 256);

        header.createCell(7, CellType.STRING).setCellValue("Text");
        sheet.setColumnWidth(7, 22 * 256);


        CellStyle headerStyle = workbook.createCellStyle();
        Font headerFont = workbook.createFont();
        headerFont.setBold(true);
        headerStyle.setFont(headerFont);
        header.cellIterator().forEachRemaining(cell -> cell.setCellStyle(headerStyle));
    }

    private void createTransaktioner(SXSSFWorkbook workbook, List<BookExcel> books) {
        SXSSFSheet sheet = workbook.getSheetAt(0);
        int rownum = 5;

        CellStyle currencyStyle = workbook.createCellStyle();
        DataFormat currencyFormat = workbook.createDataFormat();
        currencyStyle.setDataFormat(currencyFormat.getFormat("#,##0.00"));

        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");

        for (BookExcel book : books) {
            Row row = sheet.createRow(rownum++);

            row.createCell(0, CellType.STRING).setCellValue(book.getName());
            row.createCell(2, CellType.NUMERIC).setCellValue(book.getPeriod().toInteger());
            row.createCell(5, CellType.STRING).setCellValue(formatter.format(book.getBookPublishDate()));
            row.createCell(6, CellType.STRING).setCellValue(book.getDescription());
            row.createCell(7, CellType.STRING).setCellValue(book.getText());

            Cell belopp = row.createCell(10, CellType.NUMERIC);
            belopp.setCellValue(t.getBelopp().getAmount().doubleValue());
            belopp.setCellStyle(currencyStyle);

            Cell moms = row.createCell(11, CellType.NUMERIC);
            moms.setCellValue(t.getMoms().getAmount().doubleValue());
            moms.setCellStyle(currencyStyle);

            row.createCell(12, CellType.STRING).setCellValue(t.getVerifikationsnummer());
        }
    }
}
```

The BookExcelDto and repository layer

```sh

public class BookExcel {
    //Attributes, Constructors, Getter and setter
}

public interface BookRepository {
    List<BookExcel> findBooks(long bookstoresId, List<Long> writersId, Period startPeriod, Period stopPeriod);
}


@Repository
public class BookRepositoryImpl implements BookRepository {
    @Autowired
    private EntityManager em;

    public List<BookExcel> findBooks(long bookstoresId, List<Long> writersId, Period startPeriod, Period stopPeriod) {

        String query =
                "select book.name, bookstore.name, writer.name, book.publish_date" +
                "from book book " +
                "join bookstore bookstore     on bookstore.id = book.fk_bookstore " +
                "join writer writer           on writer.id = book.fk_writer " +
                "where bookstore.id = :bookstoresId " +
                "and book.publish_date >= :startPeriod " +
                "and book.publish_date <= :stopPeriod " +
                (isNotEmpty(writersId) ? "and (writer.id in :writersId) " : "") +
                "order by book.name, book.publish_date, book.id";

        Query nativeQuery = em.createNativeQuery(query);
        nativeQuery.setParameter("bookstoresId", bookstoresId);
        nativeQuery.setParameter("writersId", writersId);
        nativeQuery.setParameter("startPeriod", Integer.valueOf(startPeriod.getValue()));
        nativeQuery.setParameter("stopPeriod", Integer.valueOf(stopPeriod.getValue()));
        if (isNotEmpty(writersId)) {
            nativeQuery.setParameter("writersId", writersId);


        List<Object[]> resultList = nativeQuery.getResultList();
        List<BookExcel> Books = new ArrayList<>();

        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");

        resultList.forEach(objects -> {
            BookExcel book = new BookExcel(
                    JdbcMappingUtils.toString(objects[0]),
                    JdbcMappingUtils.toString(objects[1]),
                    JdbcMappingUtils.toString(objects[3]),
                    LocalDate.parse(JdbcMappingUtils.toString(objects[4]), formatter)
            );

            Books.add(book);
        });

        return Books;
    }
}
```


**Free Software, Hell Yeah!**

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)

   [se]: <https://superuser.com/questions/366468/what-is-the-maximum-allowed-rows-in-a-microsoft-excel-xls-or-xlsx>

