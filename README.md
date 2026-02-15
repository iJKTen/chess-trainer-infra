# Chess Trainer Infrastructure

AWS CloudFormation infrastructure for [chess.jaik.me](https://chess.jaik.me) — a React-based chess trainer app.

## Architecture

```
Route 53 (chess.jaik.me) → CloudFront (OAC) → S3 (private bucket)
```

### CI/CD Pipeline

```
GitHub (iJKTen/chess-trainer) → CodePipeline → CodeBuild (build) → CodeBuild (deploy) → S3 + CloudFront invalidation
```

## Resources

| Resource | Purpose |
|---|---|
| **S3WebsiteBucket** | Hosts built static assets (private, versioned) |
| **PipelineArtifactBucket** | Stores CodePipeline/CodeBuild artifacts (7-day expiry) |
| **CloudFrontDistribution** | CDN with custom domain, HTTPS, and SPA error handling |
| **CloudFrontOAC** | Origin Access Control for secure S3 access via SigV4 |
| **ChessTrainerPipeline** | CodePipeline with Source → Build → Deploy stages |
| **BuildProject** | CodeBuild project — `npm install` + `npm run build` (Node 20) |
| **DeployProject** | CodeBuild project — S3 sync with cache headers + CloudFront invalidation |
| **DomainRecord** | Route 53 A record aliased to CloudFront |

## Cross-Stack Dependencies

This stack imports values from other stacks:

- `GlobalResourcesStack-GitHubConnectionArn` — CodeStar connection for GitHub
- `JaikMeWildCard-CertificateArn` — ACM wildcard certificate for HTTPS
- `JaiK-HostedZoneId` — Route 53 hosted zone

## Deployment

This stack is deployed via [CloudFormation Git Sync](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/git-sync.html). Pushing changes to the repository automatically updates the stack.

## Caching Strategy

- **Hashed assets** (JS/CSS): `max-age=31536000, public, immutable` — cached forever, hash changes on new builds
- **index.html**: `max-age=0, no-cache, no-store, must-revalidate` — always revalidated to pick up new asset references
- CloudFront invalidation (`/*`) runs after every deploy
