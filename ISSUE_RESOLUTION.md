# Issue Resolution: Inconsistent Key Type Coercion in `QueryParams`

## 📌 Metadata & Scope
* **Repository:** [encode/httpx](https://github.com/encode/httpx)
* **Status:** ✅ Resolved & Verified
* **Component:** `httpx.QueryParams`
* **Affected File:** `httpx/_urls.py`
* **Test File:** `tests/models/test_queryparams.py`

---

## 🔍 Problem Description

`httpx.QueryParams` is designed to model URL query parameters. During initialization, all mapping keys are coerced to strings via `str(k)`. Furthermore, methods such as `get_list(key)`, `set(key, value)`, and `remove(key)` all consistently stringified the incoming `key` using `str(key)`.

However:
1. `__contains__(key)` directly checked `key in self._dict`.
2. `__getitem__(key)` directly accessed `self._dict[key]`.
3. `get(key, default)` checked `if key in self._dict:`.

Because `self._dict` has string keys, passing non-string keys (such as integer keys `{123: "val"}`) resulted in:
* `123 in q` evaluating to `False` (even though `q` was initialized with `{123: "val"}`).
* `q[123]` raising an unhandled `KeyError: 123`.
* `q.get(123)` returning `None` instead of `"val"`.
* In contrast, `q.get_list(123)` returned `["val"]`, creating a confusing inconsistency across the API.

---

## 🛠️ Root Cause

In `httpx/_urls.py`:

```python
# PREVIOUS CODE
def get(self, key: typing.Any, default: typing.Any = None) -> typing.Any:
    if key in self._dict:
        return self._dict[str(key)][0]
    return default

def __getitem__(self, key: typing.Any) -> str:
    return self._dict[key][0]

def __contains__(self, key: typing.Any) -> bool:
    return key in self._dict
```

The keys stored in `self._dict` were stringified at initialization time, but lookups in `get`, `__getitem__`, and `__contains__` did not stringify `key` before looking it up in `self._dict`.

---

## 🧪 Test Case Reproduction

Added `test_queryparam_non_string_keys` in [`tests/models/test_queryparams.py`](file:///Users/apple/Desktop/jmd/httpx/tests/models/test_queryparams.py):

```python
def test_queryparam_non_string_keys():
    q = httpx.QueryParams({123: "val", 456: ["a", "b"]})

    # Test key presence
    assert 123 in q
    assert "123" in q
    assert 999 not in q

    # Test __getitem__
    assert q[123] == "val"
    assert q["123"] == "val"

    # Test get()
    assert q.get(123) == "val"
    assert q.get("123") == "val"
    assert q.get(999, default="missing") == "missing"

    # Test get_list()
    assert q.get_list(456) == ["a", "b"]
    assert q.get_list("456") == ["a", "b"]
```

Before the fix, running pytest failed with:
```bash
E AssertionError: assert 123 in QueryParams('123=val&456=a&456=b')
tests/models/test_queryparams.py:143: AssertionError
```

---

## 💡 Solution

Updated `get()`, `__getitem__()`, and `__contains__()` in `httpx/_urls.py` to ensure `key` is converted using `str(key)` before membership checks or indexing:

```python
def get(self, key: typing.Any, default: typing.Any = None) -> typing.Any:
    key_str = str(key)
    if key_str in self._dict:
        return self._dict[key_str][0]
    return default

def __getitem__(self, key: typing.Any) -> str:
    return self._dict[str(key)][0]

def __contains__(self, key: typing.Any) -> bool:
    return str(key) in self._dict
```

---

## ✅ Verification
* **Pytest:** `15 passed in 0.04s`
* **Linter (ruff):** `All checks passed!`
