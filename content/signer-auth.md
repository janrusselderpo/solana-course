---
title: Signer Authorization
objectives:
- Explain the security risks associated with not performing appropriate signer checks
- Implement signer checks using long-form Rust
- Implement signer checks using Anchor’s `Signer` type
- Implement signer checks using Anchor’s `#[account(signer)]` constraint
---

# TL;DR

- Gamitin ang **Signer Checks** upang i-verify na ang mga partikular na account ay lumagda sa isang transaksyon. Kung walang naaangkop na pag-check ng lumagda, maaaring maisagawa ng mga account ang mga instruction hindi sila dapat pahintulutang gawin.
- Para magpatupad ng signer check sa Rust, tingnan lang kung ang property ng `is_signer` ng account ay `true`
    
    ```rust
    if !ctx.accounts.authority.is_signer {
    	return Err(ProgramError::MissingRequiredSignature.into());
    }
    ```
    
- Sa Anchor, maaari mong gamitin ang **`Signer`** na uri ng account sa struct ng account validation upang awtomatikong magsagawa ang Anchor ng signer check sa isang partikular na account
- Ang Anchor ay mayroon ding account constraint na awtomatikong magbe-verify na ang isang partikular na account ay pumirma ng isang transaksyon

# Pangkalahatang-ideya

Ginagamit ang mga signer check para i-verify na pinahintulutan ng may-ari ng isang account ang isang transaksyon. Kung walang signer check, ang mga pagpapatakbo na ang execution ay dapat na limitado sa mga partikular na account lamang ang posibleng gawin ng anumang account. Sa pinakamasamang sitwasyon, maaari itong magresulta sa mga wallet na ganap na maubos ng mga umaatake na pumasa sa anumang account na gusto nila sa isang instruction.

### Nawawalang Signer Check

Ang halimbawa sa ibaba ay nagpapakita ng sobrang pinasimple na bersyon ng isang instruction na nag-a-update sa field ng `awtoridad` na nakaimbak sa isang account ng program.

Pansinin na ang field ng `authority` sa struct ng validation ng account na `UpdateAuthority` ay may uri ng `AccountInfo`. Sa Anchor, ang uri ng account na `AccountInfo` ay nagpapahiwatig na walang mga pagcheck na ginawa sa account bago ang execution ng instruction.

Bagama't ang na `autorithy` ay ginagamit upang patunayan ang `awtoridad` na account na ipinasa sa instruction ay tumutugma sa field ng `awtoridad` na nakaimbak sa `vault` account, walang check upang i-verify ang `authority` account na pinahintulutan ang transaksyon.

Nangangahulugan ito na maipapasa lang ng isang attacker ang pampublikong key ng `authority` account at ang kanilang sariling public key bilang ang `new_authority` account upang muling italaga ang kanilang sarili bilang bagong awtoridad ng `vault` account. Sa puntong iyon, maaari silang makipag-ugnayan sa program bilang bagong awtoridad.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod insecure_update{
    use super::*;
        ...
        pub fn update_authority(ctx: Context<UpdateAuthority>) -> Result<()> {
        ctx.accounts.vault.authority = ctx.accounts.new_authority.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAuthority<'info> {
   #[account(
        mut,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    pub new_authority: AccountInfo<'info>,
    pub authority: AccountInfo<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

### Magdagdag ng mga check ng awtorisasyon ng lumagda

Ang kailangan mo lang gawin para ma-validate na ang `authority` account na nilagdaan ay magdagdag ng signer check sa loob ng instruction. Nangangahulugan lamang iyon ng pagsuri kung ang `authority.is_signer` ay `true`, at nagbabalik ng error na `MissingRequiredSignature` kung `false`.

```tsx
if !ctx.accounts.authority.is_signer {
    return Err(ProgramError::MissingRequiredSignature.into());
}
```

Sa pamamagitan ng pagdaragdag ng check ng lumagda, mapoproseso lamang ang instruction kung ang account ay pumasa bilang ang `autoridad` na account ay nilagdaan din ang transaksyon. Kung ang transaksyon ay hindi nilagdaan ng account na ipinasa bilang `authority` account, kung gayon ang transaksyon ay mabibigo.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod secure_update{
    use super::*;
        ...
        pub fn update_authority(ctx: Context<UpdateAuthority>) -> Result<()> {
            if !ctx.accounts.authority.is_signer {
            return Err(ProgramError::MissingRequiredSignature.into());
        }

        ctx.accounts.vault.authority = ctx.accounts.new_authority.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAuthority<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    pub new_authority: AccountInfo<'info>,
    pub authority: AccountInfo<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

### Gamitin ang uri ng account na `Signer` ng Anchor

Gayunpaman, ang paglalagay ng check na ito sa function ng instruction ay nagpapagulo sa paghihiwalay sa pagitan ng account validation at lohika ng instruction.

Sa kabutihang palad, pinapadali ng Anchor ang pagsasagawa ng mga signer check sa pamamagitan ng pagbibigay ng uri ng account na `Signer`. Baguhin lang ang uri ng `authority` account sa struct ng account validation upang maging uri ng `Signer`, at titingnan ng Anchor sa runtime na ang tinukoy na account ay isang lumagda sa transaksyon. Ito ang diskarte na karaniwang inirerekomenda natin dahil pinapayagan ka nitong paghiwalayin ang checker check mula sa lohika ng instruction.

Sa halimbawa sa ibaba, kung hindi nilagdaan ng `authority` account ang transaksyon, mabibigo ang transaksyon bago pa man maabot ang lohika ng instruction.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod secure_update{
    use super::*;
        ...
        pub fn update_authority(ctx: Context<UpdateAuthority>) -> Result<()> {
        ctx.accounts.vault.authority = ctx.accounts.new_authority.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAuthority<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    pub new_authority: AccountInfo<'info>,
    pub authority: Signer<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

Tandaan na kapag ginamit mo ang uri ng `Signer`, walang iba pang ownership o type check ang isinasagawa.

### Gamitin ang `#[account(signer)]` constraint ng Anchor

Bagama't sa karamihan ng mga kaso, ang uri ng account na `Signer` ay sapat na upang matiyak na ang isang account ay pumirma sa isang transaksyon, ang katotohanang walang ibang ownership o type check na isinasagawa ay nangangahulugan na ang account na ito ay hindi talaga magagamit para sa anumang bagay sa pagtuturo.

Dito magagamit ang `signer` *constraint*. Ang `#[account(signer)]` constraint ay nagbibigay-daan sa iyong i-verify ang account na nilagdaan ang transaksyon, habang nakakakuha din ng mga benepisyo ng paggamit sa uri ng `Account` kung gusto mo rin ng access sa pinagbabatayan na data nito.

Bilang isang halimbawa kung kailan ito magiging kapaki-pakinabang, isipin ang pagsulat ng isang instruction na inaasahan mong ma-invoke sa pamamagitan ng CPI na inaasahan na ang isa sa mga naipasa sa mga account ay parehong ******signer****** sa transaciton at isang ***********data source**************. Ang paggamit ng `Signer` na uri ng account dito ay nag-aalis ng awtomatikong deserialization at pagsuri ng uri na makukuha mo gamit ang uri ng `Account`. Ito ay parehong nakakaabala, dahil kailangan mong manual na i-deserialize ang data ng account sa lohika ng pagtuturo, at maaaring maging vulnerable ang iyong program sa pamamagitan ng hindi pagkuha ng ownership and type checking na isinagawa ayon sa uri ng `Account`.

Sa halimbawa sa ibaba, maaari mong ligtas na magsulat ng lohika upang makipag-ugnayan sa data na nakaimbak sa `authority` account habang bini-verify din na nilagdaan nito ang transaksyon.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod secure_update{
    use super::*;
        ...
        pub fn update_authority(ctx: Context<UpdateAuthority>) -> Result<()> {
        ctx.accounts.vault.authority = ctx.accounts.new_authority.key();

        // access the data stored in authority
        msg!("Total number of depositors: {}", ctx.accounts.authority.num_depositors);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAuthority<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    pub new_authority: AccountInfo<'info>,
    #[account(signer)]
    pub authority: Account<'info, AuthState>
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
#[account]
pub struct AuthState{
	amount: u64,
	num_depositors: u64,
	num_vaults: u64
}
```

# Demo

Magsanay tayo sa pamamagitan ng paglikha ng isang simpleng program upang ipakita kung paano maaaring payagan ng isang nawawalang signer check ang isang umaatake na mag-withdraw ng mga token na hindi sa kanila.

Sinisimulan ng program na ito ang isang pinasimpleng token na "vault" na account at ipinapakita kung paano maaaring payagan ng nawawalang signer check ang vault na ma-drain.

### 1. Panimula

Upang makapagsimula, i-download ang starter code mula sa `starter` branch ng [repository na ito](https://github.com/Unboxed-Software/solana-signer-auth/tree/starter). Kasama sa starter code ang isang program na may dalawang instruction at ang setup ng boilerplate para sa test file.

Ang `initialize_vault` instruction ay nagpapasimula ng dalawang bagong account: `Vault` at `TokenAccount`. Ang `Vault` account ay pasisimulan gamit ang Program Derived Address (PDA) at iimbak ang address ng isang token account at ang awtoridad ng vault. Ang awtoridad ng token account ay ang `vault` na PDA na nagbibigay-daan sa program na mag-sign para sa paglipat ng mga token.

Ang `insecure_withdraw` instruction ay maglilipat ng mga token sa token account ng `vault` account sa isang token account ng `withdraw_destination`. Gayunpaman, ang `authority` account sa `InsecureWithdraw` struct ay may uri ng `UncheckedAccount`. Ito ay isang wrapper sa paligid ng `AccountInfo` upang tahasang isaad na ang account ay hindi naka-check.

Kung walang signer check, kahit sino ay maaaring magbigay ng pampublikong key ng `authority` account na tumutugma sa `authority` na naka-store sa `vault` account at ang `insecure_withdraw` na instruction ay patuloy na ipoproseso.

Bagama't ito ay medyo gawa-gawa na ang anumang DeFi program na may vault ay magiging mas sopistikado kaysa dito, ipapakita nito kung paano ang kakulangan ng signer check ay maaaring magresulta sa pag-withdraw ng mga token ng maling partido.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod signer_authorization {
    use super::*;

    pub fn initialize_vault(ctx: Context<InitializeVault>) -> Result<()> {
        ctx.accounts.vault.token_account = ctx.accounts.token_account.key();
        ctx.accounts.vault.authority = ctx.accounts.authority.key();
        Ok(())
    }

    pub fn insecure_withdraw(ctx: Context<InsecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeVault<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 32,
        seeds = [b"vault"],
        bump
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        init,
        payer = authority,
        token::mint = mint,
        token::authority = vault,
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}

#[derive(Accounts)]
pub struct InsecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
        has_one = token_account,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    #[account(mut)]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    /// CHECK: demo missing signer check
    pub authority: UncheckedAccount<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

### 2. Subukan ang `insecure_withdraw` na instruction

Kasama sa test file ang code para i-invoke ang `initialize_vault` pagtuturo gamit ang `wallet` bilang `authority` sa vault. Ang code ay nagbibigay ng 100 token sa `vault` token account. Ayon sa teorya, ang key ng `wallet` ay dapat na ang tanging maaaring mag-withdraw ng 100 token mula sa vault.

Ngayon, magdagdag tayo ng pagsubok para ma-invoke ang `insecure_withdraw` sa program para ipakita na ang kasalukuyang bersyon ng program ay nagbibigay-daan sa isang third party na talagang bawiin ang 100 token na iyon.

Sa pagsubok, gagamitin pa rin natin ang pampublikong key ng `wallet` bilang `authority` account, ngunit gagamit kami ng ibang keypair para lagdaan at ipadala ang transaksyon.

```tsx
describe("signer-authorization", () => {
    ...
    it("Insecure withdraw", async () => {
    const tx = await program.methods
      .insecureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenAccount.publicKey,
        withdrawDestination: withdrawDestinationFake,
        authority: wallet.publicKey,
      })
      .transaction()

    await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])

    const balance = await connection.getTokenAccountBalance(
      tokenAccount.publicKey
    )
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

Patakbuhin ang `anchor test` upang makita na ang parehong mga transaksyon ay matagumpay na makukumpleto.

```bash
signer-authorization
  ✔ Initialize Vault (810ms)
  ✔ Insecure withdraw  (405ms)
```

Dahil walang signer check para sa `authority` account, ang `insecure_withdraw` na instruction ay maglilipat ng mga token mula sa `vault` token account sa `withdrawDestinationFake` token account hangga't ang pampublikong key ng`authority` account ay tumutugma sa publiko key na naka-store sa authority field ng `vault` account. Maliwanag, ang `insecure_withdraw` instruction ay kasing insecure gaya ng iminumungkahi ng pangalan.

### 3. Magdagdag ng `secure_withdraw` instruction

Ayusin natin ang problema sa isang bagong instruction na tinatawag na `secure_withdraw`. Magiging kapareho ang instruction ito sa `insecure_withdraw` instruction, maliban kung gagamitin natin ang uri ng `Signer` sa struct ng Mga Account upang i-validate ang `authority` account sa `SecureWithdraw` na struct. Kung ang `authority` account ay hindi isang lumagda sa transaksyon, inaasahan nating mabibigo ang transaksyon at magbabalik ng error.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod signer_authorization {
    use super::*;
    ...
    pub fn secure_withdraw(ctx: Context<SecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
        has_one = token_account,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    #[account(mut)]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}
```

### 4. Subukan ang `secure_withdraw` instruction

Habang nakalagay ang instruction, bumalik sa test file para subukan ang `secure_withdraw` instruction. Gawin ang `secure_withdraw` instruction, gamit muli ang pampublikong key ng `wallet` bilang `authority` account at ang keypair ng `withdrawDestinationFake` bilang ang pumirma at destinasyon ng withdraw. Dahil na-validate ang `authority` account gamit ang uri ng `Signer`, inaasahan nating mabibigo ang transaksyon sa checker check at magbabalik ng error.

```tsx
describe("signer-authorization", () => {
    ...
	it("Secure withdraw", async () => {
    try {
      const tx = await program.methods
        .secureWithdraw()
        .accounts({
          vault: vaultPDA,
          tokenAccount: tokenAccount.publicKey,
          withdrawDestination: withdrawDestinationFake,
          authority: wallet.publicKey,
        })
        .transaction()

      await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])
    } catch (err) {
      expect(err)
      console.log(err)
    }
  })
})
```

Patakbuhin ang `anchor test` upang makita na ang transaksyon ay magbabalik na ngayon ng signature verification error.

```bash
Error: Signature verification failed
```

Ayan yun! Ito ay isang medyo simpleng bagay na dapat iwasan, ngunit hindi kapani-paniwalang mahalaga. Siguraduhing palaging pag-isipan kung sino ang dapat na magpapahintulot sa mga instruction at tiyaking ang bawat isa ay lumagda sa transaksyon.

Kung gusto mong tingnan ang panghuling code ng solusyon, mahahanap mo ito sa sangay ng `solusyon` ng [imbakan](https://github.com/Unboxed-Software/solana-signer-auth/tree/solution) .

# Hamon

Sa puntong ito ng kurso, umaasa tayo nagsimula kang gumawa ng mga program at proyekto sa labas ng Mga Demo at Hamon na ibinigay sa mga araling ito. Para dito at sa natitirang mga aralin tungkol sa mga kahinaan sa seguridad, ang Hamon para sa bawat aralin ay ang pag-audit ng sarili mong code para sa kahinaan sa seguridad na tinalakay sa aralin.

Bilang kahalili, maaari kang makahanap ng mga open source na program upang i-audit. Maraming mga program ang maaari mong tingnan. Ang isang magandang simula kung hindi mo iniisip ang pagsisid sa katutubong Rust ay ang [mga program ng SPL](https://github.com/solana-labs/solana-program-library).

Kaya para sa araling ito, tingnan ang isang program (sa iyo man o isa na nahanap mo online) at i-audit ito para sa mga check ng lumagda. Kung makakita ka ng bug sa program ng ibang tao, mangyaring alertuhan sila! Kung makakita ka ng bug sa sarili mong program, siguraduhing i-patch ito kaagad.
