
# elastic_search_laravel_2021

## Elastic Search ver 7  with Laravel 8 

### composer require elasticsearch/elasticsearch
### composer require laravel/breeze --dev

## For Dashboard UI

### php artisan breeze:install
### npm install
### npm run dev
### php artisan migrate

### artisan make:model -mf Article
`code`
Schema::create('articles', function (Blueprint $table) {
     $table->id();
     $table->string('title');
     $table->text('body');
     $table->json('tags');
     $table->timestamps();
 }); 

<?php

namespace App\Models;

use App\Search\Searchable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Article extends Model
{
    use HasFactory;
    use Searchable;

    protected $casts = [
        'tags' => 'json',
    ];
}


<?php
 namespace Database\Seeders;
 use App\Models\Article;
 use Illuminate\Database\Seeder;
 class DatabaseSeeder extends Seeder
 {
     public function run()
     {
         Article::factory()->times(50)->create();
     }
 } 

<?php
 namespace Database\Factories;
 use App\Models\Article;
 use Illuminate\Database\Eloquent\Factories\Factory;
 class ArticleFactory extends Factory
 {
     protected $model = Article::class;
     public function definition()
     {
         return [
             'title' => $this->faker->sentence(),
             'body' => $this->faker->text(),
             'tags' => collect(['php', 'ruby', 'java', 'javascript', 'bash'])
                 ->random(2)
                 ->values()
                 ->all(),
         ];
     }
 } 
### Update view dashboard.php 
<x-app-layout>
     <x-slot name="header">
         <h2 class="font-semibold text-xl text-gray-800 leading-tight">
             {{ __('Articles') }} <span class="text-gray-400">({{ $articles->count() }})</span>
         </h2>
     </x-slot>
     <div class="py-12">
         <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
             <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                 <div class="p-6 bg-white border-b border-gray-200">
                     <form action="{{ route('dashboard') }}" method="get" class="pb-4">
                         <div class="form-group">
                             <x-input
                                 type="text"
                                 name="q"
                                 class="form-control"
                                 placeholder="Search..."
                                 value="{{ request('q') }}"
                             />
                         </div>
                     </form>
                     @if (request()->has('q'))
                         <p class="text-sm">Using search: <strong>"{{ request('q') }}"</strong>. <a class="border-b border-indigo-800 text-indigo-800" href="{{ route('dashboard') }}">Clear filters</a></p>
                     @endif
                     <div class="mt-8 space-y-8">
                         @forelse ($articles as $article)
                             <article class="space-y-1">
                                 <h2 class="font-semibold text-2xl">{{ $article->title }}</h2>
                                 <p class="m-0">{{ $article->body }}</body>
                                 <div>
                                     @foreach ($article->tags as $tag)
                                         <span class="text-xs px-2 py-1 rounded bg-indigo-50 text-indigo-500">{{ $tag}}</span>
                                     @endforeach
                                 </div>
                             </article>
                         @empty
                             <p>No articles found</p>
                         @endforelse
                     </div>
                 </div>
             </div>
         </div>
     </div>
 </x-app-layout> 
<?php

namespace App\Articles;

use Illuminate\Database\Eloquent\Collection;

interface SearchRepository
{
    public function search(string $query): Collection;
}

<?php
 namespace App\Articles;
 use App\Models\Article;
 use Illuminate\Database\Eloquent\Collection;
 class EloquentSearchRepository implements SearchRepository
 {
     public function search(string $term): Collection
     {
         return Article::query()
             ->where(fn ($query) => (
                 $query->where('body', 'LIKE', "%{$term}%")
                     ->orWhere('title', 'LIKE', "%{$term}%")
             ))
             ->get();
     }
 } 

use App\Articles\SearchRepository;
 Route::get('/dashboard', function (SearchRepository $searchRepository) {
     return view('dashboard', [
         'articles' => request()->has('q')
             ? $searchRepository->search(request('q'))
             : App\Models\Article::all(),
     ]);
 })->middleware(['auth'])->name('dashboard'); 

<?php

namespace App\Search;

use Elasticsearch\Client;

class ElasticsearchObserver
{
    public function __construct(private Client $elasticsearchClient)
    {
        // ...
    }

    public function saved($model)
    {
        $model->elasticSearchIndex($this->elasticsearchClient);
    }

    public function deleted($model)
    {
        $model->elasticSearchDelete($this->elasticsearchClient);
    }
}
<?php

namespace App\Search;

use Elasticsearch\Client;

trait Searchable
{
    public static function bootSearchable()
    {
        if (config('services.search.enabled')) {
            static::observe(ElasticsearchObserver::class);
        }
    }

    public function elasticsearchIndex(Client $elasticsearchClient)
    {
        $elasticsearchClient->index([
            'index' => $this->getTable(),
            'type' => '_doc',
            'id' => $this->getKey(),
            'body' => $this->toElasticsearchDocumentArray(),
        ]);
    }

    public function elasticsearchDelete(Client $elasticsearchClient)
    {
        $elasticsearchClient->delete([
            'index' => $this->getTable(),
            'type' => '_doc',
            'id' => $this->getKey(),
        ]);
    }

    abstract public function toElasticsearchDocumentArray(): array;
}

<?php

namespace App\Articles;

use App\Article;
use Elasticsearch\Client;
use Illuminate\Support\Arr;
use Illuminate\Database\Eloquent\Collection;

class ElasticsearchRepository implements ArticlesRepository
{
    /** @var \Elasticsearch\Client */
    private $elasticsearch;

    public function __construct(Client $elasticsearch)
    {
        $this->elasticsearch = $elasticsearch;
    }

    public function search(string $query = ''): Collection
    {
        $items = $this->searchOnElasticsearch($query);

        return $this->buildCollection($items);
    }

    private function searchOnElasticsearch(string $query = ''): array
    {
        $model = new Article;

        $items = $this->elasticsearch->search([
            'index' => $model->getSearchIndex(),
            'type' => $model->getSearchType(),
            'body' => [
                'query' => [
                    'multi_match' => [
                        'fields' => ['title^5', 'body', 'tags'],
                        'query' => $query,
                    ],
                ],
            ],
        ]);

        return $items;
    }

    private function buildCollection(array $items): Collection
    {
        $ids = Arr::pluck($items['hits']['hits'], '_id');

        return Article::findMany($ids)
            ->sortBy(function ($article) use ($ids) {
                return array_search($article->getKey(), $ids);
            });
    }
}

<?php

namespace App\Providers;

use App\Articles;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->bind(Articles\ArticlesRepository::class, function ($app) {
            // This is useful in case we want to turn-off our
            // search cluster or when deploying the search
            // to a live, running application at first.
            if (! config('services.search.enabled')) {
                return new Articles\EloquentRepository();
            }

            return new Articles\ElasticsearchRepository(
                $app->make(Client::class)
            );
        });
    }
}

<?php

namespace App\Providers;

use App\Articles;
use Elasticsearch\Client;
use Elasticsearch\ClientBuilder;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->bind(Articles\ArticlesRepository::class, function () {
            // This is useful in case we want to turn-off our
            // search cluster or when deploying the search
            // to a live, running application at first.
            if (! config('services.search.enabled')) {
                return new Articles\EloquentRepository();
            }

            return new Articles\ElasticsearchRepository(
                $app->make(Client::class)
            );
        });

        $this->bindSearchClient();
    }

    private function bindSearchClient()
    {
        $this->app->bind(Client::class, function ($app) {
            return ClientBuilder::create()
                ->setHosts($app['config']->get('services.search.hosts'))
                ->build();
        });
    }
}
### Update config /services.php
<?php

return [
    // ...
    'search' => [
        'enabled' => env('ELASTICSEARCH_ENABLED', false),
        'hosts' => explode(',', env('ELASTICSEARCH_HOSTS')),
    ],
];

### Add to .ENV file
ELASTICSEARCH_ENABLED=true
ELASTICSEARCH_HOSTS="localhost:9200"

<?php

namespace App\Console\Commands;

use App\Article;
use Elasticsearch\Client;
use Illuminate\Console\Command;

class ReindexCommand extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'search:reindex';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Indexes all articles to Elasticsearch';

    /** @var \Elasticsearch\Client */
    private $elasticsearch;

    public function __construct(Client $elasticsearch)
    {
        parent::__construct();

        $this->elasticsearch = $elasticsearch;
    }

    public function handle()
    {
        $this->info('Indexing all articles. This might take a while...');

        foreach (Article::cursor() as $article)
        {
            $this->elasticsearch->index([
                'index' => $article->getSearchIndex(),
                'type' => $article->getSearchType(),
                'id' => $article->getKey(),
                'body' => $article->toSearchArray(),
            ]);

            // PHPUnit-style feedback
            $this->output->write('.');
        }

        $this->info("\nDone!");
    }
}

### CLI
php artisan make:command ReindexCommand --command="search:reindex"

### Advance
<?php

'query' => [
    'multi_match' => [
        'fields' => ['title^5', 'body', 'tags'],
        'query' => $query,
    ],
],








