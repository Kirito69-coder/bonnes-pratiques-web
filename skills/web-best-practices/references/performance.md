# Performance checklist

This reference lists performance items to verify during a review.

## Loading and feedback

- Loading states are shown for slow operations.
- Buttons are disabled during submission to reduce duplicate actions.

## Caching

- HTTP caching headers are used where appropriate (`Cache-Control`, `ETag`, `Last-Modified`).
- Application-level caching is used for frequently accessed, low-volatility data when relevant.

## Network and payloads

- JS, CSS and images are compressed and reasonably sized in production.
- Requests and responses are limited to necessary data.
- File upload limits are configured where relevant.

## Rendering and responsive behavior

- Pages are usable on mobile and desktop without horizontal scrolling.
- Animations and decorative effects are controlled and not excessive.

## Database and queries

- Indexes exist on frequently queried columns where relevant.
- Pagination is used for large result sets.
- Unnecessary queries and repeated work are reduced.

## Load and availability

- Load testing is considered for expected traffic levels.
- Monitoring is configured to detect slowdowns and outages.

## References

- Performance tooling varies by stack; use Lighthouse, WebPageTest, `ab`, `siege`, or similar tools where appropriate.
- CDN and static hosting can be used for static content when appropriate.

## Notes

- Performance targets should reflect the expected audience and stack.
- Measurements should be repeated after changes, not based on a single check.
