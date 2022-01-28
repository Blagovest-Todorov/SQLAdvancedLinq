# SQLAdvancedLinq
advanced Ling for queries

///
using BookShopDB.Data;
using BookShopDB.Initializer;
using BookShopDB.Model;
using BookShopDB.Model.Enums;
using Microsoft.EntityFrameworkCore;
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using System.Text;
namespace BookShopDB
{
    class Program
    {
        static void Main(string[] args)
        {
            var db = new BookShopContext();

            db.Database.EnsureDeleted(); // WE DELETE THE OLD db IF EXISTS
            db.Database.EnsureCreated(); ///WE create the DB with the new model
            DbInitializer.ResetDatabase(db);

            //Console.WriteLine(GetBooksByAgeRestriction( db, "Minor")); Task 2
            //Console.WriteLine(GetGoldenBooks(db)); Task 3

            //
            // var test = "fdsfd".MyContains("FD", true);
            //Console.WriteLine(test);
            Console.WriteLine(GetBooksByCategory(db, "honor my drama"));
            Console.WriteLine(GetBooksReleasedBefore(db, "12-04-1992"));
            Console.WriteLine(GetAuthorNamesEndingIn(db, "e"));
            Console.WriteLine(GetBooksByAuthor(db, "R"));
            Console.WriteLine(CountBooks(db, 10));
            Console.WriteLine(CountCopiesByAuthor(db));
            Console.WriteLine(GetTotalProfitByCategory(db));
            Console.WriteLine(GetMostRecentBooks(db));
            IncreasePrices(db); /// We call only the void mehtod, we cant print void mehtos , has not result
            Console.WriteLine(RemoveBooks(db));
        }

        public static int RemoveBooks(BookShopContext context)
        {
            var books = context.Books
                 .Where(b => b.Copies < 4200)
                 .ToList(); //We take the books that have less than  4200 Copies

            context.Books.RemoveRange(books); //We remove from DB set, DataBase Table Books, the picked //collection of books

            var result = context.SaveChanges();
            return result ; /// The DB returns the changed objects into the DB, id the count of deleted Obj.

            //can be done this way  too
            //context.SaveChanges();
            //return books.Count;
        }
        public static void IncreasePrices(BookShopContext context)
        {
            var books = context.Books
                 .Where(b => b.ReleaseDate.Value.Year < 2010)
                 .ToList();

            foreach (var book in books)
            {
                book.Price += 5;
            }

            context.SaveChanges();
        }

        public static string GetMostRecentBooks(BookShopContext context)
        {
            var categoryBooks = context.Categories
                 .Select(c => new
                 {
                     c.Name,
                     ThreeBooksByCat = c.CategoryBooks
                     .Select(b => new
                     {
                         b.Book.Title,
                         b.Book.ReleaseDate.Value
                     })
                     .OrderByDescending(b => b.Value)
                     .Take(3)
                     .ToArray()
                 })
                 .OrderBy(c => c.Name)
                 .ToArray();

            var sb = new StringBuilder();

            foreach (var category in categoryBooks)
            {
                sb.AppendLine($"--{category.Name}");

                foreach (var book in category.ThreeBooksByCat)
                {
                    sb.AppendLine($"{book.Title} + {book.Value.Year}");
                }
            }

            return sb.ToString().TrimEnd();


        }
        public static string GetTotalProfitByCategory(BookShopContext context)
        {
            var categories = context.Categories
                .Select(c => new
                {
                    c.Name,
                    Profit = c.CategoryBooks.Sum(b => b.Book.Copies * b.Book.Price),
                })
                .OrderByDescending(c => c.Profit)
                .ThenBy(c => c.Name)
                .ToArray();

            var result = string.Join(Environment.NewLine, categories.Select(c => $"{c.Name} ${c.Profit:F2}"));

            return result;
        }

        public static string CountCopiesByAuthor(BookShopContext context)
        {
            var allAuthors = context.Authors
                .Select(a => new
                {
                    authorName = a.FirstName + " " + a.LastName,
                    totalBookCopies = a.Books.Sum(b => b.Copies),

                })
                .OrderByDescending(a => a.totalBookCopies)
                .ToArray();

            var result = string.Join(Environment.NewLine,
                allAuthors.Select(a => $"{a.authorName} - {a.totalBookCopies}"));

            return result;

        }
        public static string GetBooksByCategory(BookShopContext context, string input)
        {
            var categories = input
                .Split(' ', StringSplitOptions.RemoveEmptyEntries)
                .Select(x => x.ToLower())
                .ToArray();

            //var listTitleOfBooks = context.Books
            //     .Include(x => x.BookCategories)
            //     .ThenInclude( x => x.Category)
            //     .ToArray()
            //     .Where(b => b.BookCategories
            //     .Any(category => categories.Contains(category.Category.Name.ToLower())))
            //     .OrderBy(title =>  title)
            //     .ToArray();

            //Solution 2
            //List<string> booksTitles = new List<string>();
            //var allBooks = context.Books
            //    .Include(x => x.BookCategories)
            //    .ThenInclude(x =>x.Category)
            //    .OrderBy(x => x.Title)
            //    .ToList();

            //foreach (var book in allBooks)
            //{
            //    foreach (var bookCategory in book.BookCategories)
            //    {
            //        if (categories.Contains(bookCategory.Category.Name.ToLower()))
            //        {
            //            booksTitles.Add(book.Title);
            //        }
            //    }
            //}

            //var result = string.Join(Environment.NewLine, booksTitles);

            var allBooksCategories = context.BooksCategories
                .Where(bc => bc.categories.Contains(bc.Category.Name.ToLower()))
                .Select(book => book.Books.Title)
                .OrderBy(title => title)
                .ToArray();


            var result = string.Join(Environment.NewLine, allBooksCategories);

            return result;

            // string result = Print(allBookCategories);

        }

        public static int CountBooks(BookShopContext context, int lengthCheck)
        {
            var booksByLength = context.Books
                .Where(b => b.Title.Length > lengthCheck)
                .ToArray();

            int result = booksByLength.Count();

            return result;
        }

        public static string GetAuthorNamesEndingIn(BookShopContext context,
            string input)
        {
            var authors = context.Authors
                 .Where(a => EF.Functions.Like(a.FirstName, $"%{input}"))
                 .Select(a => new
                 {
                     a.FirstName,
                     a.LastName,
                 })
                 .OrderBy(a => a.FirstName)
                 .ThenBy(a => a.LastName)
                 .ToArray();

            var result = string.Join(Environment.NewLine,
                authors.Select(a => $"{a.FirstName} {a.LastName}"));

            return result;
        }

        public static string GetBooksByAuthor(BookShopContext context, string input)
        {
            var booksByAuthor = context.Books
              .Where(b => EF.Functions.Like(b.Author.LastName, $"{input}%"))
              .Select(b => new
              {
                  AuthorName = b.Author.FirstName + " " + b.Author.LastName,
                  b.Title,
                  b.BookId
              })
              .OrderBy(b => b.BookId)
              .ToArray();

            var result = string.Join(Environment.NewLine, booksByAuthor
                .Select(b => $"{b.Title} ({b.AuthorName})"));

            return result;
        }

        public static string GetBooksReleasedBefore(BookShopContext context, string date)
        {
            var targetDate = DateTime.ParseExact(date, "dd-MM-yyyy",
                CultureInfo.InvariantCulture);

            var books = context.Books
                .Where(b => b.ReleaseDate.Value < targetDate)
                .Select(b => new
                {
                    b.Title,
                    b.EditionType,
                    b.Price,
                    b.ReleaseDate.Value,
                })
                .OrderByDescending(b => b.Value)
                .ToArray();

            var result = string.Join(Environment.NewLine,
                     books.Select(b => $"{b.Title} - {b.EditionType} - ${b.Price:F2}"));

            return result;
        }

        //private static string Print(object[] allBookCategories, string separator = "\r\n") 
        //{
        //    return string.Join(separator, allBookCategories);
        //}
        public static string GetBooksNotReleasedIn(BookShopContext context, int year)
        {
            var booksNotReleasedInYear = context.Books
                 .Where(b => b.ReleaseDate.Value.Year != year)
                 .Select(b => new
                 {
                     b.Title,
                     b.BookId
                 })
                 .OrderBy(b => b.BookId)
                 .ToArray();

            var result = string.Join(Environment.NewLine,
                $"{booksNotReleasedInYear.Select(b => $"{b.Title}") }");

            return result;
        }

        public static string GetBooksByPrice(BookShopContext context)
        {
            var books = context.Books
                  .Where(book => book.Price > 40)
                  .Select(book => new
                  {
                      book.Title,
                      book.Price,
                  })
                  .OrderByDescending(b => b.Price)
                  .ToArray();

            //var sb = new StringBuilder();

            //foreach (var book in books)
            //{
            //    sb.AppendLine($"{book.Title} - ${book.Price:F2}");
            //}

            //return sb.ToString().TrimEnd();


            var result = string.Join(Environment.NewLine,
                books.Select(b => $"{b.Title} - ${b.Price}"));

            return result;
        }

        public static string GetGoldenBooks(BookShopContext context)
        {
            var books = context.Books
                 .Where(b => b.EditionType == EditionType.Gold && b.Copies < 5000)
                 .Select(b => new
                 {
                     b.BookId,
                     b.Title
                 })
                 .OrderBy(x => x.BookId)
                 .ToArray();

            var result = string.Join(Environment.NewLine,
                books.Select(x => x.Title));
            return result;
        }

        public static string GetBooksByAgeRestriction(BookShopContext context, string command)
        {
            var ageRestriction = Enum.Parse<AgeRestriction>(command, true);
            var booksTitles = context.Books
                 .Where(books => books.AgeRestriction == ageRestriction)
                 .Select(book => book.Title)
                 .OrderBy(title => title)
                 .ToArray();

            var result = string.Join(Environment.NewLine, booksTitles);

            return result;

        }

    }
}
