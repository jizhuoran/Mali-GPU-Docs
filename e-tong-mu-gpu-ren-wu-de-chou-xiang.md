# 阿童木：GPU任务的抽象

如果你生于大陆

## Getting Super Powers

Becoming a super hero is a fairly straight forward process:

```
typedef struct base_jd_atom_v2 {
	u64 jc;			    /**< job-chain GPU address */
	struct base_jd_udata udata;		    /**< user data */
	u64 extres_list;	    /**< list of external resources */
	u16 nr_extres;			    /**< nr of external resources or JIT allocations */
	u16 compat_core_req;	            /**< core requirements which correspond to the legacy support for UK 10.2 */
	struct base_dependency pre_dep[2];  /**< pre-dependencies, one need to use SETTER function to assign this field,
	this is done in order to reduce possibility of improper assigment of a dependency field */
	base_atom_id atom_number;	    /**< unique number to identify the atom */
	base_jd_prio prio;                  /**< Atom priority. Refer to @ref base_jd_prio for more details */
	u8 device_nr;			    /**< coregroup when BASE_JD_REQ_SPECIFIC_COHERENT_GROUP specified */
	u8 padding[1];
	base_jd_core_req core_req;          /**< core requirements */
} base_jd_atom_v2;
```

{% hint style="info" %}
 Super-powers are granted randomly so please submit an issue if you're not happy with yours.
{% endhint %}

Once you're strong enough, save the world:

```
// Ain't no code for that yet, sorry
echo 'You got to trust me on this, I saved the world'
```



