---
title: Vicissitudes of ELF symbol resolution
---

Was reading [dl-lookup.c] the other day.  The dynamic loader first searches a symbol by name; it then invokes `check_match`
to see if the candidate satisfies other requirements, specifically the version.  Below is an excerpt from `check_match`;
`version` is the required version of an undefined symbol; `map` is the mmap'd file from the which the candidate solution comes.

[dl-lookup.c]: https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/dl-lookup.c;h=68ecc6179f6085475d25e9796d4d6dc00465cfb3#l57

```c
  const ElfW(Half) *verstab = map->l_versyms;
  if (version != NULL)
    {
      if (__glibc_unlikely (verstab == NULL))
	{
	  /* We need a versioned symbol but haven't found any.  If
	     this is the object which is referenced in the verneed
	     entry it is a bug in the library since a symbol must
	     not simply disappear.

	     It would also be a bug in the object since it means that
	     the list of required versions is incomplete and so the
	     tests in dl-version.c haven't found a problem.*/
	  assert (version->filename == NULL
		  || ! _dl_name_match_p (version->filename, map));

	  /* Otherwise we accept the symbol.  */
	}
      else
	{
	  /* We can match the version information or use the
	     default one if it is not hidden.  */
	  ElfW(Half) ndx = verstab[symidx] & 0x7fff;
	  if ((map->l_versions[ndx].hash != version->hash
	       || strcmp (map->l_versions[ndx].name, version->name))
	      && (version->hidden || map->l_versions[ndx].hash
		  || (verstab[symidx] & 0x8000)))
	    /* It's not the version we want.  */
	    return NULL;
	}
    }
```

Let's try to restate the `else` branch, in which both the reference and the candidate are versioned:
```c
if (the version does not match && something else to boot)
  skip the candidate;
```

It seems that the comment and the code do not quite match.
I couldn't help but ask questions:

* If the version doesn't match, shouldn't this be enough to skip the candidate?
  Or can we resolve `func@V1` to `func@V2`?  I hope not.

* On the other hand, if the version does match, the `if` condition is short-circuited
  and the candidate must be accepted.  The check for `version->hidden` is ineffective.

* Hmm, `version->hidden` actually reads "hidden reference", not "hidden definition".
  Hidden references, quite a misnomer.

* The check for `map->l_versions[ndx].hash` being non-zero looks suspicious.
  The hash value can be 0 just by chance, which contributes to not skipping the candidate.
