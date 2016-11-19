# All GitHub Commit Emails

5.8 million unique GitHub commit emails (`git config user.email`) extracted from <https://www.githubarchive.org> from 2011-02-12 to 2015-12-31.

## Update 2016-08-11

Arfon https://github.com/arfon confirmed to me that emails were removed via:

- https://github.com/igrigorik/githubarchive.org/issues/112
- https://github.com/igrigorik/githubarchive.org/commit/fd53d3a80fd07289581541cc99446d2dce36c770
- https://github.com/igrigorik/githubarchive.org/commit/c9ae11426e5bcc30fe15617d009dfc602697ecde

## Update 2016-08-03

I've checked it today, and the emails seem to have been removed from GitHub Archive: the either null, or of form `<username sha>@<host>`.

## Update 2016-06-03

GitHub has asked me to take the emails down, and since they are also talking to GitHub Archive and GHTorrent, I took the emails down in good faith. GitHub Archive data was still available.

<support@github.com> 2016-06-03 email:

> Hi Ciro,

> I'm reaching out on behalf of GitHub's Support team. A number of users have contacted us with questions about the following repository: https://github.com/cirosantilli/all-github-commit-emails.

> We read through the repository's README, and appreciate your point about how easy it can be for spammers to scrape users' email addresses. We also appreciate that you've brought this to the community's (and our) attention. We thought you'd like to know we've been working with GitHub Archive and GHTorrent to scrub this data from their archives, and that both organizations have agreed to refrain from collecting this data in the future. Additionally, we'll be working with other research organizations to encourage them to comply with our policies on data minimization.

> In keeping with these efforts to better protect our users' privacy, we're asking you to take down your all-github-commit-emails repository. We'd like for you to remove it voluntarily. However, because it does violate our policies on user privacy and spam, we will remove it after one week if you do not.

> Please let me know if you have any questions.

> Best,
Elizabeth

My reply:

> Hi there.

> Were you thinking about this before, or was it I that brought it to your attention?

> Can we do it like this: I force push the actual emails out (already done), keep the README intact, and then you tell the engineering team to do a git gc to remove old commits from your APIs? This way I get to keep my heard earned stars and URL. The extraction method itself is trivial once you know the words "GitHub Archive extract commit emails".

> I'll expect a future blog post on the matter from GitHub when you've got things sorted out, clarifying / quoting your data scraping policy, and saying that GitHub Archive and GHTorrent  were not compliant. Will you forbid mass scrapping entirely, or just remove the emails? Will you modify the current events API as well?

> Please state somewhere official (above blog post or some repository a la https://github.com/github/dmca ) that my repository was taken down and why. If you found my contribution considerable, please mention it on the blog post.

> :-)

## About

Goal: let spamees find how their email leaked via Google. Spammers can get this data any time since it is public.

Please don't email me privately to take this down / remove your email from the list.

If you have a new argument, open a public issue.

Even better, tell GitHub, GitHub Archive, and <http://ghtorrent.org/> (not used here) to drop this data instead.

If they all do that, I will also take this / your email down.

Otherwise, it makes no sense to take this down, since this data is still easily extracted from the source.

GitHub has a setting to use a dummy email for web UI operations: <https://help.github.com/articles/keeping-your-email-address-private/> , but it does not affect visibility of commits done locally.

## About the data

Getting the commit email of a particular user is trivial through the API as explained at: <http://stackoverflow.com/a/32456486/895245> , so it is not much of a use case here, so usernames are not included in this data.

GitHub Archive started scraping in 2011-02-12 so older commits are not considered with the method.

In 2014-12-31, GitHub started using the new Events API.

Data is pushed daily to Google Big Query, and we will update this yearly with all the commits of the previous year.

This data is not shown on the GitHub web interface, but it is of course public because it can be seen after cloning.

GitHub also makes this data available on the `PushEvent` of the GitHub events API https://developer.github.com/v3/activity/events/types/#pushevent which GitHub Archive uses to export to a Google BigQuery table.

## The queries

Download the query data as explained at: <http://stackoverflow.com/questions/18493533/google-bigquery-download-all-data/37274820#37274820>

Extract data up to 2014-12-31

    SELECT payload_commit_email
    FROM [githubarchive:github.timeline]
    WHERE type = 'PushEvent'
    GROUP BY payload_commit_email
    ORDER BY payload_commit_email ASC

Extract data starting from 2015-01-01:

    SELECT JSON_EXTRACT(payload, '$.commits[0].author.email')
    FROM (
        TABLE_DATE_RANGE([githubarchive:day.events_],
            TIMESTAMP('2015-01-01'),
            TIMESTAMP('2015-01-02')
        ))
    WHERE type = 'PushEvent'

TODO: it would have been more intelligent to `GROUP BY` to only select unique values, and also do more cleaning on the server. Untested:

    SELECT JSON_EXTRACT_SCALAR(payload, '$.commits[0].author.email')
        AS email
    FROM (
        TABLE_DATE_RANGE([githubarchive:day.events_],
            TIMESTAMP('2015-01-01'),
            TIMESTAMP('2015-01-02')
        ))
    WHERE
        type = 'PushEvent'
        AND email <> ''
    GROUP BY email
    ORDER BY email

The above query does not work, says `email` is not a field of the table.

This would reduce the output size by an order of magnitude.

TODO: extract all emails of a given push. We currently only extract the first one at `commits[0]`. Many JSON path implementations accept `[*]`, but BigQuery does not: <http://stackoverflow.com/questions/28719880/how-to-get-all-values-of-an-attribute-of-json-array-with-jsonpath-bigquery-in-bi> 99% percent of the time it's the same email however.

## Processing the data

-   Clean up a bit if not done on the query:

        cat * | sed -E '/^$/d' | sort -u > emails-big

-   Merge data from the two queries:

        sort -u emails-old emails-new > emails-big

-   Split into multiple files:

        split -a4 -C150k -d emails-big emails/

    GitHub limits:

    -   hard limit: 100M per file, larger cannot be pushed

    -   web UI show limit:

        - TODO file size
        - 1000 files per directory

    TODO: split data further into subdirectories: `00/00`, `00/01`, ... `99/99` to make loading faster on GitHub <http://superuser.com/questions/443972/using-coreutils-split-file-into-pieces-to-different-directories>

## Data mining

Count emails:

    wc -l *

Most frequent hostnames:

    cat * | sed -E 's/.*@(.*)$/\1/' | sort | uniq -c | sort -n | tail -n 1000

TODO: how many emails are valid: not simple since not parsable by regex:

- <http://stackoverflow.com/questions/201323/using-a-regular-expression-to-validate-an-email-address>
- <http://stackoverflow.com/questions/8022530/python-check-for-valid-email-address>
- <http://stackoverflow.com/questions/2138701/email-check-regular-expression-with-bash-script>

Some common invalid emails

    grep -E '[^0-9a-zA-Z!#$%&'"'"'*+-/=?^_`{|}~@]' * | wc
    grep -v '@' * | wc

- invalid characters: <http://stackoverflow.com/questions/2049502/what-characters-are-allowed-in-email-address>
- no `@`

About 4% of the emails failed the above checks.

In particular, emails containing `<>\n` may `fsck` unhappy, and may fail to push.

For fun:

    grep 'password' *

Also contains some interesting long lines:

    grep '.\{80\}' *

## Legality

- <https://www.quora.com/unanswered/Are-version-control-e-g-Git-commit-messages-and-other-metadata-automatically-covered-by-the-same-license-as-the-project>
- <https://www.quora.com/Is-it-legal-to-sell-a-list-with-publicly-available-contact-emails>
- <https://en.wikipedia.org/wiki/CAN-SPAM_Act_of_2003>
- <https://www.avvo.com/legal-answers/can-i-copyright-my-email-address-941873.html>

## Market value

TODO: any? (if I hadn't published it)

- <http://www.5-starlists.com/freereport.html>
- <http://www.blackhatworld.com/blackhat-seo/making-money/525045-how-much-2-mil-email-list-worth.html>
- <https://www.quora.com/Where-can-I-sell-an-email-list>

## Related projects

- <https://github.com/mmautner/github-email-thief>
- <https://github.com/hodgesmr/FindGitHubEmail>
- <https://www.troyhunt.com/8-million-github-profiles-were-leaked-from-geekedins-mongodb-heres-how-to-see-yours/> the scrapper database of a company called Geekedin went public, and Troy said it was serious, But I think they don't have any data not readily available form GitHub Archive.
