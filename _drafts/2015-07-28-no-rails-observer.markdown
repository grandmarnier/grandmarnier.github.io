---
layout: post
title:  "rails-observers는 꼭 필요할까"
date:   2015-07-28 21:25:00
categories: ruby rails
---
[rails-observers](https://github.com/rails/rails-observers)는 다음 2가지 기능을 제공하는 젬이다.

1. ActiveRecord Observer
2. ActionController Sweeper

Observer는 모델 외부에서 모델의 lifecycle callback을 등록한다.
Sweeper는 observer를 컨트롤러에 연결하여 변경 사항을 보고 page/action cache를 evict하는 기능을 제공한다.
모델의 변경사항을 추적하는 등 모델의 로직과는 관계없는 코드를 파일로 분리할 수 있고
여러 모델, 컨트롤러 사이에서 공통으로 사용하는 코드들을 재사용하기 쉽게 하는 장점이 있다.

기능을 두고 보면 DRY를 쉽게 따를 수 있는 기능인데 왜 Rails 4.0 에서부터 제외되었을까?

[레일즈 4.0 릴리즈 노트](http://guides.rubyonrails.org/4_0_release_notes.html#upgrade)를 보면 이렇게 설명하고 있다.

* ActionPack page and action caching
  - manual cache eviction이 많이 필요하다.
  - Russsian Doll caching (Fragment caching)을 사용하라.
* ActiveRecord observers
  - observer는 cache eviction에만 필요하다.
  - 스파게티 코드를 만들 수 있는 여지가 크다

[Engine Yard 글](https://blog.engineyard.com/2013/rails-4-changes)의 Rails Observers, Page and Action Caching 란을 보면
마지막 부분에 대한 부가적인 설명이 있다.

> Other than cache expiration, observers tend to be abused as a dumping ground for persistence operations in many cases,
> where a callback is a better option.


즉, page/action caching은 수동으로 cache eviction을 해줘야할 구간이 많으며
이를 위해 고안된 observer는 생각했던 목적 외에 callback의 역할을 침범하는 경우가 많다고 생각한 것 같다.
실제로 observer 등록에 사용하는 메소드 이름이 callback setter와 이름이 같기 때문에 오해의 소지가 있다.

이런 이유로, 레일즈 4 에서 권장하는 방식으로 개발하면 rails-observers의 2번 기능은 필요가 없다.
그리고 1번 기능은 레일즈 4 에서 추가된 Concern으로 구현 가능하다.

{% highlight ruby %}
# rails-observers
class CommentObserver < ActiveRecord::Observer
  def after_save(comment)
    Notifications.comment("admin@do.com", "New comment was posted", comment).deliver
  end
end

# Concern Implementation
module CommentObservable
  include ActiveSupport::Concern

  def notify_comment(comment)
    Notifications.comment("admin@do.com", "New comment was posted", comment).deliver
  end

  included do
    after_save { notify_comment(self) }
  end
end

class Comment < ActiveRecord::Base
  include CommentObservable
end
{% endhighlight %}

Concern으로 구현하면 코드가 약간 늘고 callback을 모델마다 observer를 붙여야 한다.
그래도 1번 기능은 그대로 사용할 수 있고 obsering이 아닌 callback 등록의 추상화이므로
각 모델마다 어떤 callback이 등록됐는지 모델 코드에서 알 수 있어 더 낫다고 생각한다.

그러므로 레일즈 4 이상을 사용하거나 업그레이드 예정이라면 rails-observers는 필요하지 않다고 본다.
