# KawalkiKodu

Poniżej daję przykłady kodu który kiedyś napisałem, z każdym nowym projektem który robiłem starałem się coraz mocniej zaciśniać konwencję Laravela w jak najlepszy sposób. Dałem większość przykładów, nawet tych które są mniej profesjonalne ale można zauważyć elastyczność i różne sposoby rozwiązania problemu oraz sądzę, że nie ma co ukrywać faktu iż każdy uczy się z każdą linijką. Starałem się zawrzeć również przeróżne dziedziny laravaela z którymi miałem styczności. Nie mogłem niestety udostępniać security z wiadomych powodów. Metody, klasy są nie pełne i tylko do przeglądu kodu. Aktualnie większość z tego kodu, z miłą chęcią bym zrefaktorował i widzę kopiując z mojego repo jak bardzo można to poprawić. Mam nadzieję, że te to przybliży bardziej tok mojego myślenia. 

Prosty edit kontrolera

```php
    /**
     * Show the form for editing the specified resource.
     *
     * @param  Hobby  $hobby
     * @return Application|Factory|View|Response
     */
    public function edit(Hobby $hobby)
    {
        return view('admin.hobbies.edit',compact(['hobby']));
    }
```

Przykładowa złożona kolekcja resource

```php
class GroupFindResource extends JsonResource
{
    /**
     * Transform the resource into an array.
     *
     * @param  Request  $request
     * @return array
     */
    public function toArray(Request $request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'belongs' => Auth::user()->belongsToGroup($this->id),
            'author' => new UserResource(Group::find($this->id)->author),
            'count'=> Membership::where('group_id',$this->id)->where('deleted',0)->count(),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}

```

Usuwanie artykułu

```php
    /**
     * @param Article $article
     * @return bool|null
     * @throws Exception
     */
    public function destroy(Article $article)
    {
        if (auth()->user()->can('delete', Article::class)) {
            $article->Tags()->detach();
            return $article->delete();
        }
    }
```

Przykład pociętej klasy

```php
use (...)

/**
 * Class Hobby
 * @property mixed adultsMax
 * @property mixed childsMax
 * @property mixed childs
 * @property mixed adults
 * @package App\Models
 */
class Hobby extends Model
{
    use HasFactory;

    /**
     * @var string
     */
    protected $with=['tags','subscribed'];
    
    /**
     * @return BelongsToMany
     */
    public function tags()
    {
        return $this->belongsToMany(Tag::class, TagConnector::class,'hobbies_id','tags_id');
    }
     
     /**
     * @return HasMany
     */
    public function subscribed(){
       return $this->hasMany(Subscribed::class,'hobbies_id');
    }
}
```

Metoda do usuwania powiązania z tagu

```php 

    /**
     * @param  Tag  $tag
     * @return bool
     */
    public function destroyTagConnection(Tag $tag) :bool{
        $tagConnector = TagConnector::where('hobbies_id',$this->id)->where('tags_id',$tag->id)->get();
        return $tagConnector->each->delete();
    }
    
```

Relacja do tabeli nieskończonego słownika

```php 
    /**
     * @return HasManyThrough
     */
    public function dictionary()
    {
        return $this->hasManyThrough(OfferPropertiesDictionaries::class, OfferPropertiesKey::class, 'offer_id', 'id', 'id', 'offer_properties_dictionary_id')
            ->with(['properties' => function ($q) {
                $q->where('offer_id', $this->id);
            }]);
    }
```

Migracja

```php
 /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('subscribeds', function (Blueprint $table) {
            $table->id();
            $table->foreignId('hobbies_id')->references('id')->on('hobbies');
            $table->foreignId('user_id')->references('id')->on('users');
            $table->timestamps();
        });
    }
```

Alter table

```php
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::table('hobbies', function (Blueprint $table) {
            $table->foreignId('user_id')->after('slug')->references('id')->on('users');
            $table->String("group_id")->after('slug')->nullable();
            $table->String("country_currency")->after('currency')->nullable();
        });
    }
    
```

Middleware dostępu

```php

class AdminSite
{
    /**
     * Handle an incoming request.
     *
     * @param  Request  $request
     * @param  Closure  $next
     * @return mixed
     */
    public function handle(Request $request, Closure $next)
    {
        if (Auth::user()->verified == '1' && Auth::user()->hasAnyRole(['adm','smod','mod'])) {
            return $next($request);
        }
            return abort(404);
    }
}

```
Routing z middlewarem

```php
Route::group(['middleware' => 'admin.site'], function () { 
    Route::get('/administration/tags/{tag:slug}', [App\Http\Controllers\Admin\TagController::class, 'edit'])->name('admin.tags.edit');
    (...)
});

```

Część requesta

```php

 /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        switch ($this->method()) {
            case 'POST':
                return [
                    'tag' => 'required|min:1',
                    'slug' =>'required|min:1|unique:tags,slug',
                    'description' => 'required',
                    'feature' => 'required|boolean',
                    'icon' => 'required|image',
                ];
            case 'PUT':
                return [
                    'tag' => 'required|min:1',
                    'slug' =>'required|min:1|unique:tags,slug,'.$this->route()->tag->id.',id',
                    'description' => 'required',
                    'feature' => 'required|boolean',
                    'icon' => 'image',
                ];
        }
    }
```

Kiedyś na jakąś potrzebę musiałęm zrobić taką sklejkę

```php
/**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        $formRequests = [
            UserRequest::class,
            UserPropertiesRequest::class,
            UserPropertiesDictionaryRequest::class
        ];
        $rules = [];

        foreach ($formRequests as $source) {

            $rules = array_merge(
                $rules,
                (new $source)->rules($this->method())
            );
        }
        return $rules;
    }
```

```php
Article::with('Tags')->whereHas("Types", function ($q) { $q->where('types.slug', 'news'); })->where('visible', 1)->where('accepted', 1)->orderBy('created_at', 'desc')->limit(5)->get();
```

```php
/**
     * @param TagRequest $tagRequest
     * @return Application|RedirectResponse|Redirector
     */
    public function store(TagRequest $tagRequest)
    {
        if (auth()->user()->can('create', Tag::class)) {
            if(!Tag::where('tag',$tagRequest->input('tag'))->exists()){
                DB::transaction(function() use ($tagRequest) {
                    $tag = Tag::create([
                        'tag' => strtolower($tagRequest->input('tag')),
                        'color' => $tagRequest->input('color')
                    ]);
                    if ($tagRequest->input('game')) {
                        Game::create([
                            'tag_id' => $tag->id,
                            'name' => $tag->tag,
                            'page_id' => $tagRequest->input('gamePage'),
                        ]);
                    }
                });
                return redirect(route('adminTags'));
            }else{
                return redirect(route('adminTagNew'))->withErrors(__('This tag exists.'));
            }
        }
```

```php
 public function belongsToGroup(int $id){
        return $this->groups->contains('id',$id) ?  true  :  false;
    }
```

Kawałek klasy Event ustawianie atrybutu
```php
class Event extends Post
{
    protected $fillable = [...]
    protected $guarded= [...];


    public function setAllDayAttribute($value){
        $this->attributes['all_day'] = $value =='on';
    }

    public function setTimeStartDateAttribute($value){
        if(is_null($value)) return;
        $this->attributes['start_date'] = $this->attributes['start_date'].' '.$value;
    }

    public function setTimeEndDateAttribute($value){
        if(is_null($value)) return;
        $this->attributes['end_date'] = $this->attributes['end_date'].' '.$value;
    }
    
     public function comments(){
        return $this->hasMany(Comment::class,'model_id')->where('Model','Event');
     }
}

```

Polityki

```php
    /**
     * Determine whether the user can update the model.
     *
     * @param  User  $user
     * @param  Article  $article
     * @return mixed
     */
    public function update(User $user, Article $article)
    {
        return $user->hasAnyRole(['adm','smod','mod']);
    }
```

```php
    /**
     * @param User $user
     * @param Article $article
     * @return bool
     */
    public function delete(User $user, Article $article)
    {
        return $user->id === $article->user_id;
    }
```

Tutaj małe zapytanie w providers (aktualnie nie wykorzystywane nie zagłębiałem się teraz dlaczego, prawdopodobnie znalazłem już lepszy sposób na zastąpienie tego kodu)

```php
        View::composer('*', function($view)
        {
            if((!is_null(\Route::getCurrentRoute())) && (\Route::getCurrentRoute()->slug !=null ))
            $view->with('sidebarArticleCompared', Article::withAnyTag(Article::firstOrFail('slug',\Route::getCurrentRoute()->slug)->first()->tags->pluck('tag'))->orderBy('created_at','DESC')->paginate(4));
            $view->with('pagesCategory', PageCategory::with(array('pages' => function($query) { $query->select('pages.name','pages.slug','pages.visible')->where('pages.visible',1); }))->whereHas("pages", function ($query) { $query->where('pages.visible', 1); })->where('page_categories.type',1)->where('page_categories.visible',1)->orderBy('order_number')->get());
        });
```

```php
    /**
     * @return bool
     * @throws Exception
     */
    public function createDefaultImage(){
        $img = DefaultProfileImage::create($this->first_name." ".$this->last_name,512,'#5DDD67');

        $url = 'public/Users';
        if (!Storage::exists($url)){
            Storage::makeDirectory($url);
        }
        return Storage::put($url."/".$this->id.".png",$img->encode());
    }
```

**widoki/blade**

```php
<meta property="og:title" content="@if(View::hasSection('metatTitle')) @yield('metatTitle') @else @yield('title') @endif "/>
<meta property="og:description" content="@if(View::hasSection('metaDescription')) @yield('metaDescription') @else Vrkadia strona o szczęściu w wirtualnej rzeczywistości @endif "/>
<meta property="og:image" content=" @if(View::hasSection('metaImage')) @yield('metaImage') @else {{asset('images/preview.png')}} @endif "/>
```

Widok z komponentem

```php
@extends('layouts.app')
@section('title',ucfirst($tag->tag))
@section('content')
	
	<div class="row">
		@foreach($articles as $article)
			<div class="col-md-4">
				<x-article-box :article="$article"/>
			</div>
		@endforeach
	</div>
@endsection
``` 

```php
<div class="col-md-10">
				
				<h2 class="nk-product-title h3">{{$user->dictionary()->where('slug','discord')->first()->properties->value ?? ''}}</h2>
				{{--				<div class="nk-gap-1"></div>--}}
				<div class="nk-product-description">
					{{$user->dictionary()->where('slug','describe')->first()->properties->value ?? ''}}
				</div>
				<div class="nk-gap-1"></div>
				@foreach($user->dictionary as $dictionar)
					@if(
					strtolower($dictionar->slug) === 'describe' ||
					strtolower($dictionar->slug) === 'discord'	||
					strtolower($dictionar->slug) === 'avatar-img'
					)
						@continue
					@endif()
					<label for="{{$dictionar->slug}}">{{__($dictionar->name)}}:</label>
					{{$dictionar->properties->value}}
					<div class="nk-gap"></div>
				@endforeach
				@can('update',$user)
					<a href="{{route('userEdit',['user'=>$user])}}" class="float-right btn btn-vrkadia-edit">Edytuj</a>
				@endcan
				
				<div class="nk-gap-1"></div>
			</div>
```

jedna z konfiguracji filesystem 
```php
        'tags' => [
            'driver' => 'local',
            'root' => storage_path('app/public/tags'),
            'url' => env('APP_URL').'/storage/tags',
            'visibility' => 'public',
        ],
```

```php
Hobby::withAnyTag($tagsFilter)
            ->where('visible', 1)
            ->where('accepted', 1)
            ->when(!empty($request->input('location')) ,function ($query, $req) use ($request) {
                return $query->where('country','like','%'.$request->input('location').'%');
            })
            ->when(!empty($request->input('date')) ,function ($query, $req) use ($dates) {
                return $query->whereBetween('start',[date('Y-m-d', strtotime($dates[0])),date('Y-m-d',strtotime($dates[1]) )])->OrwhereBetween('end',[date('Y-m-d', strtotime($dates[0])),date('Y-m-d',strtotime($dates[1]) )]);
            })
            ->when(!empty($request->input('adults')) ,function ($query, $req) use ($request) {
                return $query->whereBetween('adultsMax',[0,$request->input('adults')]);
            })
            ->when(!empty($request->input('childrens')) ,function ($query, $req) use ($request) {
                return $query->whereBetween('childsMax',[0,$request->input('childrens')]);
            })
            ->orderBy($sort,$ASCDESC)
            ->withCount('tags')
            ->get();
```
