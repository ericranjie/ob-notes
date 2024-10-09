

```cpp
namespace std {
  template<class Key,
           class T,
           class Hash = hash<Key>,
           class Pred = equal_to<Key>,
           class Allocator = allocator<pair<const Key, T>>>
  class unordered_multimap {
  public:
    // types
    using key_type             = Key;
    using mapped_type          = T;
    using value_type           = pair<const Key, T>;
    using hasher               = Hash;
    using key_equal            = Pred;
    using allocator_type       = Allocator;
    using pointer              = typename allocator_traits<Allocator>::pointer;
    using const_pointer        = typename allocator_traits<Allocator>::const_pointer;
    using reference            = value_type&;
    using const_reference      = const value_type&;
    using size_type            = /* implementation-defined */;
    using difference_type      = /* implementation-defined */;
 
    using iterator             = /* implementation-defined */;
    using const_iterator       = /* implementation-defined */;
    using local_iterator       = /* implementation-defined */;
    using const_local_iterator = /* implementation-defined */;
    using node_type            = /* unspecified */;
 
    // construct/copy/destroy
    unordered_multimap();
    explicit unordered_multimap(size_type n,
                                const hasher& hf = hasher(),
                                const key_equal& eql = key_equal(),
                                const allocator_type& a = allocator_type());
    template<class InputIt>
      unordered_multimap(InputIt f, InputIt l,
                         size_type n = /* see description */,
                         const hasher& hf = hasher(),
                         const key_equal& eql = key_equal(),
                         const allocator_type& a = allocator_type());
    template<container-compatible-range<value_type> R>
      unordered_multimap(from_range_t, R&& rg,
                         size_type n = /* see description */,
                         const hasher& hf = hasher(),
                         const key_equal& eql = key_equal(),
                         const allocator_type& a = allocator_type());
    unordered_multimap(const unordered_multimap&);
    unordered_multimap(unordered_multimap&&);
    explicit unordered_multimap(const Allocator&);
    unordered_multimap(const unordered_multimap&, const type_identity_t<Allocator>&);
    unordered_multimap(unordered_multimap&&, const type_identity_t<Allocator>&);
    unordered_multimap(initializer_list<value_type> il,
                       size_type n = /* see description */,
                       const hasher& hf = hasher(),
                       const key_equal& eql = key_equal(),
                       const allocator_type& a = allocator_type());
    unordered_multimap(size_type n, const allocator_type& a)
      : unordered_multimap(n, hasher(), key_equal(), a) { }
    unordered_multimap(size_type n, const hasher& hf, const allocator_type& a)
      : unordered_multimap(n, hf, key_equal(), a) { }
    template<class InputIt>
      unordered_multimap(InputIt f, InputIt l, size_type n, const allocator_type& a)
        : unordered_multimap(f, l, n, hasher(), key_equal(), a) { }
    template<class InputIt>
      unordered_multimap(InputIt f, InputIt l, size_type n, const hasher& hf,
                         const allocator_type& a)
        : unordered_multimap(f, l, n, hf, key_equal(), a) { }
  template<container-compatible-range<value_type> R>
    unordered_multimap(from_range_t, R&& rg, size_type n, const allocator_type& a)
      : unordered_multimap(from_range, std::forward<R>(rg),
                           n, hasher(), key_equal(), a) { }
  template<container-compatible-range<value_type> R>
    unordered_multimap(from_range_t, R&& rg, size_type n, const hasher& hf,
                       const allocator_type& a)
      : unordered_multimap(from_range, std::forward<R>(rg), n, hf, key_equal(), a) { }
    unordered_multimap(initializer_list<value_type> il, size_type n,
                       const allocator_type& a)
      : unordered_multimap(il, n, hasher(), key_equal(), a) { }
    unordered_multimap(initializer_list<value_type> il, size_type n, const hasher& hf,
                       const allocator_type& a)
      : unordered_multimap(il, n, hf, key_equal(), a) { }
    ~unordered_multimap();
    unordered_multimap& operator=(const unordered_multimap&);
    unordered_multimap& operator=(unordered_multimap&&)
      noexcept(allocator_traits<Allocator>::is_always_equal::value &&
               is_nothrow_move_assignable_v<Hash> &&
               is_nothrow_move_assignable_v<Pred>);
    unordered_multimap& operator=(initializer_list<value_type>);
    allocator_type get_allocator() const noexcept;
 
    // iterators
    iterator       begin() noexcept;
    const_iterator begin() const noexcept;
    iterator       end() noexcept;
    const_iterator end() const noexcept;
    const_iterator cbegin() const noexcept;
    const_iterator cend() const noexcept;
 
    // capacity
    [[nodiscard]] bool empty() const noexcept;
    size_type size() const noexcept;
    size_type max_size() const noexcept;
 
    // modifiers
    template<class... Args> iterator emplace(Args&&... args);
    template<class... Args> iterator emplace_hint(const_iterator position,
                                                  Args&&... args);
    iterator insert(const value_type& obj);
    iterator insert(value_type&& obj);
    template<class P> iterator insert(P&& obj);
    iterator insert(const_iterator hint, const value_type& obj);
    iterator insert(const_iterator hint, value_type&& obj);
    template<class P> iterator insert(const_iterator hint, P&& obj);
    template<class InputIt> void insert(InputIt first, InputIt last);
    template<container-compatible-range<value_type> R>
      void insert_range(R&& rg);
    void insert(initializer_list<value_type>);
 
    node_type extract(const_iterator position);
    node_type extract(const key_type& x);
    template<class K> node_type extract(K&& x);
    iterator insert(node_type&& nh);
    iterator insert(const_iterator hint, node_type&& nh);
 
    iterator  erase(iterator position);
    iterator  erase(const_iterator position);
    size_type erase(const key_type& k);
    template<class K> size_type erase(K&& x);
    iterator  erase(const_iterator first, const_iterator last);
    void      swap(unordered_multimap&)
      noexcept(allocator_traits<Allocator>::is_always_equal::value &&
               is_nothrow_swappable_v<Hash> &&
               is_nothrow_swappable_v<Pred>);
    void      clear() noexcept;
 
    template<class H2, class P2>
      void merge(unordered_multimap<Key, T, H2, P2, Allocator>& source);
    template<class H2, class P2>
      void merge(unordered_multimap<Key, T, H2, P2, Allocator>&& source);
    template<class H2, class P2>
      void merge(unordered_map<Key, T, H2, P2, Allocator>& source);
    template<class H2, class P2>
      void merge(unordered_map<Key, T, H2, P2, Allocator>&& source);
 
    // observers
    hasher hash_function() const;
    key_equal key_eq() const;
 
    // map operations
    iterator         find(const key_type& k);
    const_iterator   find(const key_type& k) const;
    template<class K>
      iterator       find(const K& k);
    template<class K>
      const_iterator find(const K& k) const;
    size_type        count(const key_type& k) const;
    template<class K>
      size_type      count(const K& k) const;
    bool             contains(const key_type& k) const;
    template<class K>
      bool           contains(const K& k) const;
    pair<iterator, iterator>               equal_range(const key_type& k);
    pair<const_iterator, const_iterator>   equal_range(const key_type& k) const;
    template<class K>
      pair<iterator, iterator>             equal_range(const K& k);
    template<class K>
      pair<const_iterator, const_iterator> equal_range(const K& k) const;
 
    // bucket interface
    size_type bucket_count() const noexcept;
    size_type max_bucket_count() const noexcept;
    size_type bucket_size(size_type n) const;
    size_type bucket(const key_type& k) const;
    local_iterator begin(size_type n);
    const_local_iterator begin(size_type n) const;
    local_iterator end(size_type n);
    const_local_iterator end(size_type n) const;
    const_local_iterator cbegin(size_type n) const;
    const_local_iterator cend(size_type n) const;
 
    // hash policy
    float load_factor() const noexcept;
    float max_load_factor() const noexcept;
    void max_load_factor(float z);
    void rehash(size_type n);
    void reserve(size_type n);
  };
 
  template<class InputIt,
           class Hash = hash<__iter_key_type<InputIt>>,
           class Pred = equal_to<__iter_key_type<InputIt>>,
           class Allocator = allocator<__iter_to_alloc_type<InputIt>>>
    unordered_multimap(InputIt, InputIt,
                       typename /* see description */::size_type = /* see description */,
                       Hash = Hash(), Pred = Pred(), Allocator = Allocator())
      -> unordered_multimap<__iter_key_type<InputIt>, __iter_mapped_type<InputIt>,
                            Hash, Pred, Allocator>;
 
  template<ranges::input_range R,
           class Hash = hash<__range_key_type<R>>,
           class Pred = equal_to<__range_key_type<R>>,
           class Allocator = allocator<__range_to_alloc_type<R>>>
    unordered_multimap(from_range_t, R&&,
                       typename /* see description */::size_type = /* see description */,
                       Hash = Hash(), Pred = Pred(), Allocator = Allocator())
      -> unordered_multimap<__range_key_type<R>, __range_mapped_type<R>, Hash, Pred,
                            Allocator>;
 
  template<class Key, class T, class Hash = hash<Key>,
           class Pred = equal_to<Key>, class Allocator = allocator<pair<const Key, T>>>
    unordered_multimap(initializer_list<pair<Key, T>>,
                       typename /* see description */::size_type = /* see description */,
                       Hash = Hash(), Pred = Pred(), Allocator = Allocator())
      -> unordered_multimap<Key, T, Hash, Pred, Allocator>;
 
  template<class InputIt, class Allocator>
    unordered_multimap(InputIt, InputIt, typename /* see description */::size_type,
                       Allocator)
      -> unordered_multimap<__iter_key_type<InputIt>, __iter_mapped_type<InputIt>,
                            hash<__iter_key_type<InputIt>>,
                            equal_to<__iter_key_type<InputIt>>, Allocator>;
 
  template<class InputIt, class Allocator>
    unordered_multimap(InputIt, InputIt, Allocator)
      -> unordered_multimap<__iter_key_type<InputIt>, __iter_mapped_type<InputIt>,
                            hash<__iter_key_type<InputIt>>,
                            equal_to<__iter_key_type<InputIt>>, Allocator>;
 
  template<class InputIt, class Hash, class Allocator>
    unordered_multimap(InputIt, InputIt, typename /* see description */::size_type, Hash,
                       Allocator)
      -> unordered_multimap<__iter_key_type<InputIt>, __iter_mapped_type<InputIt>, Hash,
                            equal_to<__iter_key_type<InputIt>>, Allocator>;
 
  template<ranges::input_range R, class Allocator>
    unordered_multimap(from_range_t, R&&, typename /* see description */::size_type,
                       Allocator)
      -> unordered_multimap<__range_key_type<R>, __range_mapped_type<R>,
                            hash<__range_key_type<R>>,
                            equal_to<__range_key_type<R>>, Allocator>;
 
  template<ranges::input_range R, class Allocator>
    unordered_multimap(from_range_t, R&&, Allocator)
      -> unordered_multimap<__range_key_type<R>, __range_mapped_type<R>,
                            hash<__range_key_type<R>>,
                            equal_to<__range_key_type<R>>, Allocator>;
 
  template<ranges::input_range R, class Hash, class Allocator>
    unordered_multimap(from_range_t, R&&, typename /* see description */::size_type, Hash,
                       Allocator)
      -> unordered_multimap<__range_key_type<R>, __range_mapped_type<R>, Hash,
                            equal_to<__range_key_type<R>>, Allocator>;
 
  template<class Key, class T, class Allocator>
    unordered_multimap(initializer_list<pair<Key, T>>,
                       typename /* see description */::size_type,
                       Allocator)
      -> unordered_multimap<Key, T, hash<Key>, equal_to<Key>, Allocator>;
 
  template<class Key, class T, class Allocator>
    unordered_multimap(initializer_list<pair<Key, T>>, Allocator)
      -> unordered_multimap<Key, T, hash<Key>, equal_to<Key>, Allocator>;
 
  template<class Key, class T, class Hash, class Allocator>
    unordered_multimap(initializer_list<pair<Key, T>>,
                       typename /* see description */::size_type,
                       Hash, Allocator)
      -> unordered_multimap<Key, T, Hash, equal_to<Key>, Allocator>;
}
```