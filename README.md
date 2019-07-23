namespace Blog.Controllers
{
    using System.Threading.Tasks;
    using AutoMapper;
    using Infrastructure;
    using Microsoft.AspNetCore.Authorization;
    using Microsoft.AspNetCore.Mvc;
    using Models;
    using Services;

    public class ArticlesController : Controller
    {
        private readonly IArticleService articleService;
        private readonly IMapper mapper;

        public ArticlesController(
            IArticleService articleService,
            IMapper mapper)
        {
            this.articleService = articleService;
            this.mapper = mapper;
        }

        public async Task<IActionResult> All([FromQuery]int page = 1) 
            => this.View(new ArticleListingViewModel
            {
                Articles = await this.articleService.All(page),
                Total = await this.articleService.Total(),
                Page = page
            });

        public async Task<IActionResult> Details(int id)
        {
            var article = await this.articleService.Details(id);

            if (article == null)
            {
                return this.NotFound();
            }

            if (!this.User.IsAdministrator() 
                && !article.IsPublic 
                && article.Author != this.User.Identity.Name)
            {
                return this.NotFound();
            }

            return this.View(article);
        }

        [HttpGet]
        [Authorize]
        public IActionResult Create() => this.View();

        [HttpPost]
        [Authorize]
        public async Task<IActionResult> Create(ArticleFormModel article)
        {
            if (this.ModelState.IsValid)
            {
                await this.articleService.Add(article.Title, article.Content, this.User.GetId());

                this.TempData.Add(ControllerConstants.SuccessMessage, "Article created successfully it is waiting for approval!");

                return this.RedirectToAction(nameof(this.Mine));
            }

            return this.View(article);
        }

        [HttpGet]
        [Authorize]
        public async Task<IActionResult> Edit(int id)
        {
            var article = await this.articleService.Details(id);

            if (article == null || (article.Author != this.User.Identity.Name && !this.User.IsAdministrator()))
            {
                return this.NotFound();
            }

            var articleEdit = this.mapper.Map<ArticleFormModel>(article);

            return this.View(articleEdit);
        }

        [HttpPost]
        [Authorize]
        public async Task<IActionResult> Edit(int id, ArticleFormModel article)
        {
            if (!await this.articleService.IsByUser(id, this.User.GetId()) && !this.User.IsAdministrator())
            {
                return this.NotFound();
            }
            
            if (this.ModelState.IsValid)
            {
                await this.articleService.Edit(id, article.Title, article.Content);

                this.TempData.Add(ControllerConstants.SuccessMessage, "Article edited successfully and is waiting for approval!");

                return this.RedirectToAction(nameof(this.Mine));
            }

            return this.View(article);
        }
        
        [Authorize]
        public async Task<IActionResult> Delete(int id)
        {
            if (!await this.articleService.IsByUser(id, this.User.GetId()) && !this.User.IsAdministrator())
            {
                return this.NotFound();
            }

            return this.View(id);
        }
        
        [Authorize]
        public async Task<IActionResult> ConfirmDelete(int id)
        {
            if (!await this.articleService.IsByUser(id, this.User.GetId()) && !this.User.IsAdministrator())
            {
                return this.NotFound();
            }

            await this.articleService.Delete(id);
            
            this.TempData.Add(ControllerConstants.SuccessMessage, "Article deleted successfully!");

            return this.RedirectToAction(nameof(this.Mine));
        }

        [Authorize]
        public async Task<IActionResult> Mine()
        {
            var articles = await this.articleService.ByUser(this.User.GetId());

            return this.View(articles);
        }
 public class ArticleService : IArticleService
    {
        private readonly BlogDbContext db;
        private readonly IMapper mapper;
        private readonly IDateTimeProvider dateTimeProvider;

        public ArticleService(
            BlogDbContext db, 
            IMapper mapper, 
            IDateTimeProvider dateTimeProvider)
        {
            this.db = db;
            this.mapper = mapper;
            this.dateTimeProvider = dateTimeProvider;
        }

        public async Task<IEnumerable<ArticleListingServiceModel>> All(
            int page = 1,
            int pageSize = ServicesConstants.ArticlesPerPage,
            bool publicOnly = true)
            => await this.All<ArticleListingServiceModel>(page, pageSize, publicOnly);

        public async Task<IEnumerable<TModel>> All<TModel>(
            int page = 1,
            int pageSize = ServicesConstants.ArticlesPerPage,
            bool publicOnly = true)
            where TModel : class
        {
            var query = this.db.Articles.AsQueryable();

            if (publicOnly)
            {
                query = query.Where(a => a.IsPublic);
            }

            return await query
                .OrderByDescending(a => a.PublishedOn)
                .Skip((page - 1) * pageSize)
                .Take(pageSize)
                .ProjectTo<TModel>(this.mapper.ConfigurationProvider)
                .ToListAsync();
        }

        public async Task<IEnumerable<ArticleForUserListingServiceModel>> ByUser(string userId)
            => await this.db
                .Articles
                .Where(a => a.UserId == userId)
                .OrderByDescending(a => a.PublishedOn)
                .ProjectTo<ArticleForUserListingServiceModel>(this.mapper.ConfigurationProvider)
                .ToListAsync();

        public async Task<bool> IsByUser(int id, string userId)
            => await this.db
                .Articles
                .AnyAsync(a => a.Id == id && a.UserId == userId);

        public async Task<ArticleDetailsServiceModel> Details(int id)
            => await this.db
                .Articles
                .Where(a => a.Id == id)
                .ProjectTo<ArticleDetailsServiceModel>(this.mapper.ConfigurationProvider)
                .FirstOrDefaultAsync();

        public async Task<int> Total()
            => await this.db
                .Articles
                .Where(a => a.IsPublic)
                .CountAsync();

        public async Task<int> Add(string title, string content, string userId)
        {
            var article = new Article
            {
                Title = title,
                Content = content,
                UserId = userId
            };

            this.db.Articles.Add(article);

            await this.db.SaveChangesAsync();

            return article.Id;
        }

        public async Task Edit(int id, string title, string content)
        {
            var article = await this.db.Articles.FindAsync(id);

            if (article == null)
            {
                return;
            }

            article.Title = title;
            article.Content = content;
            article.IsPublic = false;

            await this.db.SaveChangesAsync();
        }

        public async Task Delete(int id)
        {
            var article = await this.db.Articles.FindAsync(id);
            this.db.Articles.Remove(article);

            await this.db.SaveChangesAsync();
        }

        public async Task ChangeVisibility(int id)
        {
            var article = await this.db.Articles.FindAsync(id);

            if (article == null)
            {
                return;
            }

            article.IsPublic = !article.IsPublic;

            if (article.PublishedOn == null)
            {
                article.PublishedOn = this.dateTimeProvider.Now();    
            }

            await this.db.SaveChangesAsync();
        }
    }
}
