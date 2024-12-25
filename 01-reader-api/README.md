## GitHub changes

The following commit was prepared with the help of the notepad.

https://github.com/blockscout/blockscout/pull/11541/commits/da3d1481c84eede82e093b948e911d92dd68eb37

## Notes

The dialog started with:

> Take a look at @reader-api  and particularly on the section "Proposed Solution". What functions from @api.ex must go to `apps/explorer/lib/explorer/chain/arbitrum/reader/api/messages.ex`?

The consequent messages:

> Yes, create the module `Explorer.Chain.Arbitrum.Reader.API.Messages` and move the functions listed above.

> What functions from @api.ex must go to `apps/explorer/lib/explorer/chain/arbitrum/reader/api/settlement.ex`?

> Yes, create the module `Explorer.Chain.Arbitrum.Reader.API.Settlement` and move the functions listed above.

> @reader-api @api.ex Create the module `Explorer.Chain.Arbitrum.Reader.API.General`.

> Take a look at "High-level Scope of Changes" in@reader-api and refactor modules listed in the item 1.

Afer the previous step manual editing of files was required since some of affected functions was refactored incorrectly. That is why it was decided to make the next steps file by file.

> Take a look at "High-level Scope of Changes" in @reader-api  and adjust the file `arbitrum.md` listed in the item 2.

> Take a look at "High-level Scope of Changes" in @reader-api   and adjust the file `common.ex` listed in the item 2.

> Take a look at "High-level Scope of Changes" in @reader-api  and adjust the file @README.md  listed in the item 2.

> Take a look at @reader-api one more time and double check that all the functions from @api.ex was moved to the modules @settlement.ex @messages.ex and @general.ex properly - as it was described in "Proposed Solution".