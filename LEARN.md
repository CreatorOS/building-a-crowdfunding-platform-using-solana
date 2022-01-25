# Building a crowdfunding platform using solana!

Solana uses Rust as the basic programming language. If you don’t know Rust already, refer to the Rust for Solana Quest (). It is a great primer to all the concepts and basics of rust you need to know to get started with writing code for Solana programs.

You also need to have a command line understanding. We’ll be operating heavily on the commandline to compile and deploy Solana programs. We recommend using Linux, Mac or WSL if you are on windows.

You must have Node installed on your machine/WSL. If you don’t, you can head over to the following link to install : [https://nodejs.org/en/download/](https://nodejs.org/en/download/)

With these prerequisites, you will be able to launch your own crowdfunding platform on Solana blockchain. You can start crowdfunding and start accepting some money for your own little project :) In this Quest we’ll cover the basics of how to write a basic program on Solana and cover some of the fundamentals that come with it.

The code we’ll be writing is to run crowdfunding campaigns. Users can request for crowdfunding, and others can send in money to them via this program we write.

Now, on to building ...

## Why on solana

We can write a crowdfunding campaign using say python or node or any of the web2 technologies we might be familiar with.

To write this webapp, you’ll need to set up the payment gateway so as to accept money. You will need to use some database that will store the information on what is the current state of the funding.

By writing a Solana Program, you don’t need to use a separate payment gateway like RazorPay or Stripe. Payment processing is a part of the Solana infrastructure itself. Also, unlike any web2 program you write, all the code is public. People can see what exactly you are doing with the money they send to your software.

In a traditional web2 crowdfunding platform like say kickstarter, you need to send money to kickstarter and trust kickstarter to honestly process the payment and disburse it to the appropriate project owner. Have you seen the code for all of this? Probably not. Even if you’ve seen the code, are you certain that that is the same code that kickstarter is actually running on their servers?

By writing code on Solana, you’re not only telling the world exactly what you are doing with their money, by giving them the source code. But, anyone can be sure that that is also the exact code that is actually being run. Their money is being used exactly according to the logic in the source code.

## Defining our data structure

Pull up your favorite code editor and start writing code alongside.

To run a crowdfunding platform, we need to keep track of how much money each project is seeking. We will also store some text that will describe the crowdfunding campaign itself.

We’ll define a public structure :

As a part of the crowdfunding platform, we’ll store the amount requested, the description of the campaign and how much has been fulfilled so far, for each campaign

```

pub struct CampaignAccount {

    pub campaign_owner: PubKey,

    pub campaign_amounts: u64,

    pub campaign_descriptions: String,

    pub campaign_fulfilled: u64,

}

```

We will create an account for each campaign. All these accounts will be managed by the logic we write. Money can be sent to these campaign accounts. That is how we will crowdfund our Campaigns. These are called Program Derived Accounts (PDAs).

## writing logic

Now that we’ve defined what our data structure is, we need to write logic that will operate on the data itself.

Every program must have an entrypoint that will be called each time some user wants to interact with the code we’ve written.

All the users will hit this one endpoint, our code should execute different logic based on the data that is sent to this endpoint. If the data says “new campaign”, it should create a new campaign. If the data says “fund campaign”, the code should execute that logic. We’ll do this with a sequence of if - else statements.

This function is called the process_instruction function

```

pub fun process_instruction (

  program_id : &Pubkey,

  accounts : &[AccountInfo],

  data :&[u8],

) -> ProgramResult

```

The program_id is the identifier for the program itself. When you want to call a program, you must also pass this id, so that solana knows which program is to be executed.

The second parameter to this function is an array of accounts that will be operated upon in this program.

Every user on Solana has a unique account. Every code that you write and deploy also has a unique account. An account can store money in a currency called Lamports. We can transfer lamports from one account to another. When you send a user some money, they can spend it however they want. If you send money to the account operated by a program, that program can use that money defined in the logic of the code itself.

So, coming back, the second parameter to our process function is the list of accounts that will be operated upon in this code. If the function is called to create a new campaign, you need to create a new account and for this campaign and send that as a parameter. If you are calling this function to fund a campaign, the array will contain the account of the funder and the account of the campaign being funded.

The data is the last parameter. The data tells what the function needs to do and all the data that is required to execute the logic.

## defining the instruction

Whatever you want to do in this program will be processed by the same function “process_instruction”. But we have multiple things that this function needs to be able to do, including creating a new campaign and adding funds to a campaign. To differentiate between the operation we want to execute, we’ll send an identifier in the data parameter.

The parameter is of type [u8] which is basically an array of bytes.

We’ll assume that the first byte of this data tells us what instruction to run

```

let (instruction_byte, all_other_bytes) = data.split_first().unwrap();

if *i nstruction_byte == 0{

  // create campaign

}

else if *instruction_byte == 1{

  // fund a campaign

}

else if *instruction_byte == 2{

  // get how much funds are left to reach the requested amount

}

else if *instruction_byte == 3{

  // withdraw all collected funds and close campaign

}

```

This is the skeleton of our program. Now all that we need to do is fill up each of those if statements. Easy peasy.

## create campaign

To create the campaign, we need to send the account info of the person creating this campaign in our accounts parameter. We will assume that one person can have only one campaign running at any point in time.

Let’s first parse the accounts.

`let iterable_accounts = &mut accounts.iter();`

This lets us fetch the accounts one after another, by iterating over the list.

`let campaign_account = next_account_info(iterable_accounts);`

The first account in the accounts array will be the campaign we want to operate upon. Either to create a new campaign or to fund an existing campaign.

This owner will also be the same person who will be able to withdraw the funds later.

The amount that this campaign is requesting and the description of the campaign will be in the data itself `rest_of_data` that we had defined in the previous subquest.

The first byte was the operation code.

The next 4 bytes will be the amount.

The rest of the bytes will be the string that describes this campaign.

```

    let amount = rest_of_data

      .get(..8)

      .and_then(|slice| slice.try_into().ok())

      .map(u64::from_le_bytes)

      .unwrap();



    let description = String::from_utf8(rest_of_data[9..].to_vec()).unwrap();

```

Now we need to create a campaign object and store it in our ProgramData HashMap. For that we need the key for this campaign.

The key for this campaign needs to be unique.

Every account has a public key and a private key. The public key, also called address, is akin to a username. The private key is something we will look at a little later, but it is similar to the password for this account.

We had earlier seen that an account can store lamports i.e money. But an account can also store data. The account for this campaign will store the data that we defined in the struct called `CampaignAccount`.

```

let mut campaign_account_data= CampaignAccount::try_from_slice(&campaign_account.data.borrow())?;

      campaign_account_data.campaign_amount = amount;

      campaign_account_data.campaign_description = description;

      campaign_account_data.campaign_fulfilled = 0;

```

Then we store it back on to the account’s data by serializing it, so that it persists.

`campaign_account_data.serialize(&mut &mut campaign_account.data.borrow_mut()[..])?;`

Return the function using `Ok(());`

## getting status of a campaign

Once a campaign is made, let’s check it’s status using instruction code 2 in our if-else skeleton.

We will assume the user who is requesting this information will send the account of the owner of the campaign using the [accounts] parameter.

Just like before

```

    let campaign_account= next_account_info(accounts_iter)?;

```

We’ll print the amount left to be collected, using msg. This would be

```

      let mut campaign_account_data = CampaignAccount::try_from_slice(&campaign_account.data.borrow())?;

      msg!("{}",campaign_account_data.campaign_amount - campaign_account_data.campaign_fulfilled);

```

Now that we’ve understood some of the core elements of Solana, let’s deploy this program and see it in action in the next Quest.

## Including the right libraries

Now that we’ve written some logic, let’s make sure we write all the other boiler plate stuff so that we’re able to actually compile the code.

We’ll import the functions we are using in our logic from a package/crate called solana_program

```

use solana_program::{

    account_info::{next_account_info, AccountInfo},

    entrypoint,

    entrypoint::ProgramResult,

    msg,

    program_error::ProgramError,

    pubkey::Pubkey,

};

```

We also need to tell the compiler to make our ProgramData datastructure to be storable. The way to store this data on to the Solana Persistent Storage, is to serialize it. We’ll use a serializer called the BorchSerializer

`use borsh::{BorshDeserialize, BorshSerialize};`

Tell the compiler that the struct ProgramData can be serialized and deserialized using this library by including the following line just above the ProgramData struct.

`#[derive(BorshSerialize, BorshDeserialize, Debug)]`

Lastly, define which function is to be called when someone is trying to interact with our program. This we do using the entrypoint function that solana_program exposes.

`entrypoint!(process_instruction);`

So your entire file would look like this:

[https://gist.github.com/madhavanmalolan/b30b47640449f92ea00e4075d63460a6](https://gist.github.com/madhavanmalolan/b30b47640449f92ea00e4075d63460a6)

## Up next

We’ve now seen how to write some code in solana.

In the two quests that follow we will,

1. Look at how to deploy and actually run this program on Solana blockchain
2. We’ll transfer money to a campaign and withdraw it when the campaign is completed
